# f2

import 'package:firebase_database/firebase_database.dart';
import 'package:fireone/real_time/add.dart';
import 'package:flutter/material.dart';
import 'package:intl/intl.dart';

class HomeScreen1 extends StatefulWidget {
  @override
  State<HomeScreen1> createState() => _HomeScreen1State();
}

class _HomeScreen1State extends State<HomeScreen1> {
  double totalTodayCalories = 0;
  List<Map> filteredEntries = [];
  double totalRangeCalories = 0;

  DateTime? fromDate;
  DateTime? toDate;

  final db = FirebaseDatabase.instance.ref();

  @override
  void initState() {
    super.initState();
    calculateTodayCalories();
  }

  Future<void> calculateTodayCalories() async {
    String today = DateFormat('yyyy-MM-dd').format(DateTime.now());
    final snapshot = await db.child('food_entries/$today').get();

    double total = 0;
    if (snapshot.exists) {
      final entries = Map<String, dynamic>.from(snapshot.value as dynamic);
      for (var entry in entries.values) {
        total += double.tryParse(entry['totalCalories'].toString()) ?? 0;
      }
    }

    setState(() {
      totalTodayCalories = total;
    });
  }

  void showQuantityDialog(Map food) {
    TextEditingController quantityController = TextEditingController();

    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text("Enter quantity (grams)"),
        content: TextField(
          controller: quantityController,
          keyboardType: TextInputType.number,
          decoration: InputDecoration(hintText: "e.g. 150"),
        ),
        actions: [
          TextButton(
            onPressed: () async {
              int quantity = int.tryParse(quantityController.text) ?? 0;
              double per100g =
                  double.tryParse(food['caloriesPer100g'].toString()) ?? 0;
              double totalCalories = (quantity * per100g) / 100;
              String today = DateFormat('yyyy-MM-dd').format(DateTime.now());

              await db.child('food_entries/$today').push().set({
                'foodName': food['name'],
                'quantity': quantity,
                'totalCalories': totalCalories
              });

              Navigator.pop(context);
              calculateTodayCalories();
            },
            child: Text("Add"),
          )
        ],
      ),
    );
  }

  void showEditEntryDialog(Map entry) {
    TextEditingController quantityController =
        TextEditingController(text: entry['quantity'].toString());

    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text("Edit Quantity for ${entry['foodName']}"),
        content: TextField(
          controller: quantityController,
          keyboardType: TextInputType.number,
          decoration: InputDecoration(hintText: "e.g. 150"),
        ),
        actions: [
          TextButton(
            onPressed: () async {
              int newQty = int.tryParse(quantityController.text) ?? 0;
              double per100g = 0;

              // Fetch original food calories per 100g
              final foodsSnap = await db.child('foods').get();
              if (foodsSnap.exists) {
                Map<String, dynamic> foods =
                    Map<String, dynamic>.from(foodsSnap.value as dynamic);
                foods.forEach((key, value) {
                  if (value['name'] == entry['foodName']) {
                    per100g =
                        double.tryParse(value['caloriesPer100g'].toString()) ??
                            0;
                  }
                });
              }

              double newTotalCalories = (newQty * per100g) / 100;

              // 🔄 Update in Firebase using date + key
              final date = entry['date'];
              final snapshot = await db.child('food_entries/$date').get();

              if (snapshot.exists) {
                final map =
                    Map<String, dynamic>.from(snapshot.value as dynamic);
                map.forEach((key, value) async {
                  if (value['foodName'] == entry['foodName'] &&
                      value['quantity'].toString() ==
                          entry['quantity'].toString() &&
                      value['totalCalories'].toString() ==
                          entry['totalCalories'].toString()) {
                    await db.child('food_entries/$date/$key').update({
                      'quantity': newQty,
                      'totalCalories': newTotalCalories,
                    });
                  }
                });
              }

              Navigator.pop(context);
              fetchEntriesByDateRange();
              calculateTodayCalories();
            },
            child: Text("Update"),
          ),
        ],
      ),
    );
  }

  void showEditDialog(Map food, String key) {
    final nameController = TextEditingController(text: food['name']);
    final caloriesController =
        TextEditingController(text: food['caloriesPer100g'].toString());

    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text("Edit Food"),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(
              controller: nameController,
              decoration: InputDecoration(labelText: "Name"),
            ),
            TextField(
              controller: caloriesController,
              keyboardType: TextInputType.number,
              decoration: InputDecoration(labelText: "Calories per 100g"),
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () async {
              String name = nameController.text.trim();
              double calories =
                  double.tryParse(caloriesController.text.trim()) ?? 0;

              await db.child('foods').child(key).update({
                'name': name,
                'caloriesPer100g': calories,
              });

              Navigator.pop(context);
              setState(() {});
            },
            child: Text("Save"),
          )
        ],
      ),
    );
  }

  Future<void> pickDate({required bool isFrom}) async {
    DateTime? picked = await showDatePicker(
      context: context,
      initialDate: DateTime.now(),
      firstDate: DateTime(2023),
      lastDate: DateTime(2100),
    );
    if (picked != null) {
      setState(() {
        if (isFrom)
          fromDate = picked;
        else
          toDate = picked;
      });
    }
  }

  Future<void> fetchEntriesByDateRange() async {
    if (fromDate == null || toDate == null) return;

    List<Map> allEntries = [];
    double rangeTotal = 0;

    for (DateTime date = fromDate!;
        date.isBefore(toDate!.add(Duration(days: 1)));
        date = date.add(Duration(days: 1))) {
      String dateStr = DateFormat('yyyy-MM-dd').format(date);
      final snapshot = await db.child('food_entries/$dateStr').get();
      if (snapshot.exists) {
        final dayEntries = Map<String, dynamic>.from(snapshot.value as dynamic);
        for (var entry in dayEntries.values) {
          double cals = double.tryParse(entry['totalCalories'].toString()) ?? 0;
          allEntries.add({
            'foodName': entry['foodName'],
            'quantity': entry['quantity'],
            'totalCalories': cals,
            'date': dateStr,
          });
          rangeTotal += cals;
        }
      }
    }

    setState(() {
      filteredEntries = allEntries;
      totalRangeCalories = rangeTotal;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          Navigator.push(
              context, MaterialPageRoute(builder: (_) => AddFoodScreen1()));
        },
        child: Icon(Icons.add),
      ),
      appBar: AppBar(title: Text("Calorie Calculator")),
      body: Column(
        children: [
          Expanded(
            child: FutureBuilder<DataSnapshot>(
              future: db.child('foods').get(),
              builder: (context, snapshot) {
                if (!snapshot.hasData)
                  return Center(child: CircularProgressIndicator());

                Map<String, dynamic> foods =
                    Map<String, dynamic>.from(snapshot.data!.value as dynamic);

                return ListView(
                  children: foods.entries.map((entry) {
                    final key = entry.key;
                    final food = entry.value;
                    return ListTile(
                      title: Text(food['name']),
                      subtitle:
                          Text("Per 100g: ${food['caloriesPer100g']} kcal"),
                      trailing: Row(
                        mainAxisSize: MainAxisSize.min,
                        children: [
                          IconButton(
                            icon: Icon(Icons.edit),
                            onPressed: () => showEditDialog(food, key),
                          ),
                          IconButton(
                            icon: Icon(Icons.add),
                            onPressed: () => showQuantityDialog(food),
                          ),
                        ],
                      ),
                    );
                  }).toList(),
                );
              },
            ),
          ),
          Divider(),
          Padding(
            padding: const EdgeInsets.all(8),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceAround,
              children: [
                TextButton(
                  onPressed: () => pickDate(isFrom: true),
                  child: Text(fromDate == null
                      ? "From"
                      : DateFormat('yyyy-MM-dd').format(fromDate!)),
                ),
                TextButton(
                  onPressed: () => pickDate(isFrom: false),
                  child: Text(toDate == null
                      ? "To"
                      : DateFormat('yyyy-MM-dd').format(toDate!)),
                ),
                ElevatedButton(
                  onPressed: fetchEntriesByDateRange,
                  child: Text("Show"),
                ),
              ],
            ),
          ),
          if (filteredEntries.isNotEmpty)
            Expanded(
              child: ListView.builder(
                itemCount: filteredEntries.length,
                itemBuilder: (context, index) {
                  var entry = filteredEntries[index];
                  return ListTile(
                    title: Text(entry['foodName']),
                    subtitle: Text(
                        "${entry['quantity']}g - ${entry['totalCalories']} kcal"),
                    // trailing: Text(entry['date']),

                    trailing: Row(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        IconButton(
                            onPressed: () {
                              showEditEntryDialog(entry);
                            },
                            icon: Icon(Icons.edit)),
                        Text(entry['date']),
                      ],
                    ),
                  );
                },
              ),
            ),
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: Column(
              children: [
                Text(
                  "Today: ${totalTodayCalories.toStringAsFixed(1)} kcal",
                  style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
                ),
                if (fromDate != null && toDate != null)
                  Text(
                    "Range: ${totalRangeCalories.toStringAsFixed(1)} kcal",
                    style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
                  ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

addfoood
import 'package:firebase_database/firebase_database.dart';
import 'package:flutter/material.dart';

class AddFoodScreen1 extends StatefulWidget {
  @override
  _AddFoodScreen1State createState() => _AddFoodScreen1State();
}

class _AddFoodScreen1State extends State<AddFoodScreen1> {
  final TextEditingController _nameController = TextEditingController();
  final TextEditingController _caloriesController = TextEditingController();
  final _formKey = GlobalKey<FormState>();

  final dbRef = FirebaseDatabase.instance.ref().child('foods');

  Future<void> addFood() async {
    if (_formKey.currentState!.validate()) {
      String name = _nameController.text.trim();
      double calories = double.tryParse(_caloriesController.text.trim()) ?? 0;

      await dbRef.push().set({
        'name': name,
        'caloriesPer100g': calories,
      });

      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Food added successfully')),
      );

      _nameController.clear();
      _caloriesController.clear();
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Add Food")),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              TextFormField(
                controller: _nameController,
                decoration: InputDecoration(labelText: 'Food Name'),
                validator: (value) =>
                    value == null || value.isEmpty ? 'Enter name' : null,
              ),
              TextFormField(
                controller: _caloriesController,
                decoration: InputDecoration(labelText: 'Calories per 100g'),
                keyboardType: TextInputType.number,
                validator: (value) =>
                    value == null || value.isEmpty ? 'Enter calories' : null,
              ),
              SizedBox(height: 20),
              ElevatedButton(
                onPressed: addFood,
                child: Text('Add Food'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
