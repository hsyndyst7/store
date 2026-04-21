# My-first-appl-cation
import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:url_launcher/url_launcher.dart';
import 'dart:io';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const ETradeApp());
}


const List<String> categories = [
  "Cilt Bakımı",
  "Makyaj",
  "Saç Bakımı",
  "Parfümeri",
  "Kişisel Hijyen",
  "Ağız Bakımı"
];


class Product {
  final String id;
  final String name;
  final String price;
  final String category; // Kategori alanı şart!
  final String? imagePath;

  Product({required this.id, required this.name, required this.price, required this.category, this.imagePath});

  factory Product.fromFirestore(DocumentSnapshot doc) {
    Map data = doc.data() as Map<String, dynamic>;
    return Product(
      id: doc.id,
      name: data['name'] ?? '',
      price: data['price'] ?? '',
      category: data['category'] ?? 'Genel',
      imagePath: data['imagePath'],
    );
  }
}


class StoreData {
  static List<Product> cart = [];
  static double calculateTotal() {
    double total = 0;
    for (var p in cart) {
      String cleanPrice = p.price.replaceAll(' TL', '').replaceAll('.', '').replaceAll(',', '');
      total += double.tryParse(cleanPrice) ?? 0;
    }
    return total;
  }
}

class ETradeApp extends StatelessWidget {
  const ETradeApp({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(primarySwatch: Colors.pink, useMaterial3: true),
      home: const LoginScreen(),
    );
  }
}


class LoginScreen extends StatelessWidget {
  const LoginScreen({super.key});

  void _checkSellerPassword(BuildContext context) {
    final passwordCtrl = TextEditingController();
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text("Satıcı Girişi"),
        content: TextField(controller: passwordCtrl, obscureText: true, decoration: const InputDecoration(labelText: "Şifre")),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: const Text("İptal")),
          ElevatedButton(onPressed: () {
            if (passwordCtrl.text == "frknztrk27") {
              Navigator.pop(context);
              Navigator.push(context, MaterialPageRoute(builder: (context) => const SellerDashboard()));
            }
          }, child: const Text("Giriş")),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Container(
        width: double.infinity,
        decoration: BoxDecoration(gradient: LinearGradient(colors: [Colors.pink.shade400, Colors.orange.shade300], begin: Alignment.topCenter)),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.auto_awesome, size: 80, color: Colors.white),
            const Text("NFM KOZMETİK", style: TextStyle(fontSize: 32, fontWeight: FontWeight.bold, color: Colors.white)),
            const SizedBox(height: 50),
            ElevatedButton(onPressed: () => Navigator.push(context, MaterialPageRoute(builder: (context) => const CustomerDashboard())), child: const Text("Müşteri Girişi")),
            const SizedBox(height: 10),
            OutlinedButton(onPressed: () => _checkSellerPassword(context), style: OutlinedButton.styleFrom(side: const BorderSide(color: Colors.white)), child: const Text("Satıcı Paneli", style: TextStyle(color: Colors.white))),
          ],
        ),
      ),
    );
  }
}


class CustomerDashboard extends StatefulWidget {
  const CustomerDashboard({super.key});
  @override
  State<CustomerDashboard> createState() => _CustomerDashboardState();
}

class _CustomerDashboardState extends State<CustomerDashboard> {
  String selectedCategory = categories[0]; // Varsayılan: Cilt Bakımı

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Kozmetik Ürünleri"), actions: [
        IconButton(icon: const Icon(Icons.shopping_cart), onPressed: () => Navigator.push(context, MaterialPageRoute(builder: (context) => const CartPage())).then((_) => setState(() {})))
      ]),
      body: Column(
        children: [
         
          SingleChildScrollView(
            scrollDirection: Axis.horizontal,
            child: Row(
              children: categories.map((cat) => Padding(
                padding: const EdgeInsets.all(5.0),
                child: ChoiceChip(
                  label: Text(cat),
                  selected: selectedCategory == cat,
                  onSelected: (bool selected) { if(selected) setState(() => selectedCategory = cat); },
                ),
              )).toList(),
            ),
          ),
          
          Expanded(
            child: StreamBuilder<QuerySnapshot>(
              stream: FirebaseFirestore.instance
                  .collection('products')
                  .where('category', isEqualTo: selectedCategory) // FİLTRE BURADA!
                  .snapshots(),
              builder: (context, snapshot) {
                if (!snapshot.hasData) return const Center(child: CircularProgressIndicator());
                var docs = snapshot.data!.docs;
                if (docs.isEmpty) return const Center(child: Text("Bu kategoride ürün yok kanka!"));
                return GridView.builder(
                  padding: const EdgeInsets.all(10),
                  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 2, childAspectRatio: 0.7, crossAxisSpacing: 10, mainAxisSpacing: 10),
                  itemCount: docs.length,
                  itemBuilder: (context, index) {
                    final p = Product.fromFirestore(docs[index]);
                    return Card(
                      child: Column(
                        children: [
                          Expanded(child: p.imagePath != null ? Image.file(File(p.imagePath!), fit: BoxFit.cover) : const Icon(Icons.image)),
                          Text(p.name, style: const TextStyle(fontWeight: FontWeight.bold)),
                          Text(p.price, style: const TextStyle(color: Colors.pink)),
                          IconButton(icon: const Icon(Icons.add_shopping_cart), onPressed: () {
                            setState(() => StoreData.cart.add(p));
                            ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text("Sepete Eklendi!")));
                          })
                        ],
                      ),
                    );
                  },
                );
              },
            ),
          ),
        ],
      ),
    );
  }
}


class SellerDashboard extends StatefulWidget {
  const SellerDashboard({super.key});
  @override
  State<SellerDashboard> createState() => _SellerDashboardState();
}

class _SellerDashboardState extends State<SellerDashboard> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Yönetim Paneli")),
      body: StreamBuilder<QuerySnapshot>(
        stream: FirebaseFirestore.instance.collection('products').snapshots(),
        builder: (context, snapshot) {
          if (!snapshot.hasData) return const Center(child: CircularProgressIndicator());
          var docs = snapshot.data!.docs;
          return ListView.builder(
            itemCount: docs.length,
            itemBuilder: (context, index) {
              final p = Product.fromFirestore(docs[index]);
              return ListTile(
                title: Text(p.name),
                subtitle: Text("${p.category} - ${p.price}"),
                trailing: IconButton(icon: const Icon(Icons.delete, color: Colors.red), onPressed: () => FirebaseFirestore.instance.collection('products').doc(p.id).delete()),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          final nameCtrl = TextEditingController();
          final priceCtrl = TextEditingController();
          String selectedCat = categories[0];
          File? selectedImg;

          await showDialog(
            context: context,
            builder: (context) => StatefulBuilder(builder: (context, setD) => AlertDialog(
              title: const Text("Ürün Ekle"),
              content: SingleChildScrollView(
                child: Column(mainAxisSize: MainAxisSize.min, children: [
                  TextButton(onPressed: () async {
                    final p = await ImagePicker().pickImage(source: ImageSource.gallery);
                    if (p != null) setD(() => selectedImg = File(p.path));
                  }, child: const Text("Resim Seç")),
                  TextField(controller: nameCtrl, decoration: const InputDecoration(labelText: "Ürün Adı")),
                  TextField(controller: priceCtrl, decoration: const InputDecoration(labelText: "Fiyat")),
                  const SizedBox(height: 10),
                  const Text("Kategori Seç:"),
                  DropdownButton<String>(
                    value: selectedCat,
                    isExpanded: true,
                    items: categories.map((cat) => DropdownMenuItem(value: cat, child: Text(cat))).toList(),
                    onChanged: (val) { if (val != null) setD(() => selectedCat = val); },
                  ),
                ]),
              ),
              actions: [
                ElevatedButton(onPressed: () {
                  FirebaseFirestore.instance.collection('products').add({
                    'name': nameCtrl.text,
                    'price': "${priceCtrl.text} TL",
                    'category': selectedCat, 
                    'imagePath': selectedImg?.path,
                  });
                  Navigator.pop(context);
                }, child: const Text("Buluta Ekle"))
              ],
            )),
          );
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}



class CartPage extends StatefulWidget {
  const CartPage({super.key});
  @override
  State<CartPage> createState() => _CartPageState();
}

class _CartPageState extends State<CartPage> {

  
  void _sendWhatsApp() async {
    String myNum = "905375029907"; 
    
   
    String msg = "NFM Kozmetik Sipariş:\n\n";
    for (var p in StoreData.cart) {
      msg += "- ${p.name} (${p.price})\n";
    }
    msg += "\n*Toplam: ${StoreData.calculateTotal().toStringAsFixed(0)} TL*";

    final Uri whatsappUrl = Uri.parse("https://wa.me/$myNum?text=${Uri.encodeComponent(msg)}");

    try {
      await launchUrl(whatsappUrl, mode: LaunchMode.externalApplication);
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text("WhatsApp açılamadı!")),
        );
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Sepetim")),
      body: Column(
        children: [
          Expanded(
            child: ListView.builder(
              itemCount: StoreData.cart.length,
              itemBuilder: (context, index) => ListTile(
                title: Text(StoreData.cart[index].name),
                subtitle: Text(StoreData.cart[index].price),
                trailing: IconButton(
                  icon: const Icon(Icons.delete), 
                  onPressed: () => setState(() => StoreData.cart.removeAt(index))
                ),
              ),
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(20),
            child: ElevatedButton(
              style: ElevatedButton.styleFrom(
                backgroundColor: Colors.green, 
                foregroundColor: Colors.white, 
                minimumSize: const Size(double.infinity, 50)
              ),
              
              onPressed: StoreData.cart.isEmpty ? null : _sendWhatsApp, 
              child: const Text("WhatsApp Sipariş Gönder"),
            ),
          )
        ],
      ),
    );
  }
}
