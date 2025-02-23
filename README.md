# Prestamos
Repositorio
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_storage/firebase_storage.dart';
import 'package:image_picker/image_picker.dart';
import 'package:url_launcher/url_launcher.dart'; // Para abrir WhatsApp y correo
import 'package:pdf/widgets.dart' as pw; // Para generar el pagaré
import 'package:path_provider/path_provider.dart'; // Para guardar el pagaré
import 'package:flutter_paypal_payment/flutter_paypal_payment.dart'; // Para PayPal
import 'dart:io';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'App de Préstamos',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        scaffoldBackgroundColor: Colors.grey[100],
        appBarTheme: AppBarTheme(
          backgroundColor: Colors.blue,
          elevation: 0,
        ),
      ),
      home: HomeScreen(),
      routes: {
        '/verificacion': (context) => VerificacionScreen(),
        '/configuracion': (context) => ConfiguracionScreen(),
        '/soporte': (context) => SoporteScreen(),
      },
    );
  }
}

class HomeScreen extends StatefulWidget {
  @override
  _HomeScreenState createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;
  final FirebaseStorage _storage = FirebaseStorage.instance;
  final ImagePicker _picker = ImagePicker();
  File? _boletaImage;
  File? _dpiImage;
  bool _verificado = false;
  double _limitePrestamo = 200.0;
  double _prestamoActual = 0.0;
  double _tasaInteres = 0.15;
  double _gastoAdministrativo = 0.05;
  int _plazoPrestamo = 30; // Plazo por defecto
  bool _esSuscriptor = false;

  Future<void> _subirDocumento(File file, String tipo) async {
    try {
      final String nombreArchivo = '${_auth.currentUser!.uid}_$tipo';
      final Reference ref = _storage.ref().child('documentos/$nombreArchivo');
      await ref.putFile(file);
      final String url = await ref.getDownloadURL();
      await _firestore.collection('verificaciones').add({
        'usuarioId': _auth.currentUser!.uid,
        'tipo': tipo,
        'url': url,
        'fecha': DateTime.now(),
        'verificado': false,
      });
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Documento subido exitosamente')),
      );
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error al subir documento: $e')),
      );
    }
  }

  Future<void> _cargarDocumentos() async {
    final XFile? boleta = await _picker.pickImage(source: ImageSource.gallery);
    final XFile? dpi = await _picker.pickImage(source: ImageSource.gallery);
    if (boleta != null && dpi != null) {
      await _subirDocumento(File(boleta.path), 'recibo');
      await _subirDocumento(File(dpi.path), 'dpi');
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Documentos enviados para verificación')),
      );
    } else {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Debes seleccionar ambos documentos')),
      );
    }
  }

  Future<void> _solicitarPrestamo(double monto) async {
    if (!_verificado) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Debes estar verificado para solicitar un préstamo')),
      );
      return;
    }

    if (monto > _limitePrestamo) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('El monto excede tu límite de préstamo')),
      );
      return;
    }

    if (_prestamoActual > 0) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Ya tienes un préstamo activo')),
      );
      return;
    }

    double intereses = monto * _tasaInteres;
    double gastos = monto * _gastoAdministrativo;
    double totalPagar = monto + intereses + gastos;

    setState(() {
      _prestamoActual = totalPagar;
    });

    await _firestore.collection('prestamos').add({
      'monto': monto,
      'intereses': intereses,
      'gastos': gastos,
      'totalPagar': totalPagar,
      'fecha': DateTime.now(),
      'plazo': _plazoPrestamo,
      'usuarioId': _auth.currentUser!.uid,
    });

    // Generar pagaré
    final pagare = await _generarPagare(monto, intereses, gastos, totalPagar);
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Préstamo aprobado. Total a pagar: Q$totalPagar')),
    );
  }

  Future<void> _suscribirse() async {
    Navigator.of(context).push(
      MaterialPageRoute(
        builder: (BuildContext context) => PaypalPayment(
          onSuccess: (payment) async {
            setState(() {
              _esSuscriptor = true;
              _tasaInteres = 0.07;
              _gastoAdministrativo = 0.0;
            });
            await _firestore.collection('usuarios').doc(_auth.currentUser!.uid).update({
              'esSuscriptor': true,
            });
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text('¡Suscripción activada! Tasa reducida al 7%')),
            );
          },
          onError: (error) {
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text('Error en el pago: $error')),
            );
          },
          onCancel: () {
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text('Pago cancelado')),
            );
          },
          currency: 'USD',
          clientId: 'TU_CLIENT_ID_DE_PAYPAL',
          secretKey: 'TU_SECRET_KEY_DE_PAYPAL',
          transactions: [
            {
              "amount": {
                "total": '50.00',
                "currency": "USD",
                "details": {
                  "subtotal": '50.00',
                  "shipping": '0.00',
                  "shipping_discount": '0.00'
                }
              },
              "description": "Suscripción mensual",
            }
          ],
        ),
      ),
    );
  }

  Future<void> _generarPagare(double monto, double intereses, double gastos, double totalPagar) async {
    final pdf = pw.Document();
    pdf.addPage(
      pw.Page(
        build: (pw.Context context) {
          return pw.Column(
            children: [
              pw.Text('Pagaré', style: pw.TextStyle(fontSize: 24, fontWeight: pw.FontWeight.bold)),
              pw.SizedBox(height: 20),
              pw.Text('Yo, ${_auth.currentUser!.email}, me comprometo a pagar la cantidad de Q$totalPagar.'),
              pw.Text('Monto del préstamo: Q$monto'),
              pw.Text('Intereses: Q$intereses'),
              pw.Text('Gastos administrativos: Q$gastos'),
              pw.Text('Plazo: $_plazoPrestamo días'),
              pw.Text('Fecha: ${DateTime.now()}'),
            ],
          );
        },
      ),
    );

    final output = await getTemporaryDirectory();
    final file = File('${output.path}/pagare.pdf');
    await file.writeAsBytes(await pdf.save());
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Pagaré generado: ${file.path}')),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('App de Préstamos'),
        actions: [
          IconButton(
            icon: Icon(Icons.verified_user),
            onPressed: () => Navigator.pushNamed(context, '/verificacion'),
          ),
          IconButton(
            icon: Icon(Icons.settings),
            onPressed: () => Navigator.pushNamed(context, '/configuracion'),
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            if (!_verificado)
              FadeIn(
                child: Card(
                  elevation: 4,
                  child: Padding(
                    padding: const EdgeInsets.all(16.0),
                    child: Column(
                      children: [
                        Text(
                          'Verificación Requerida',
                          style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
                        ),
                        SizedBox(height: 10),
                        Text(
                          'Sube tu recibo y DPI para ser verificado.',
                          style: TextStyle(fontSize: 16),
                        ),
                        SizedBox(height: 20),
                        ScaleAnimation(
                          child: ElevatedButton(
                            onPressed: _cargarDocumentos,
                            child: Text('Subir Documentos'),
                          ),
                        ),
                      ],
                    ),
                  ),
                ),
              ),
            if (_verificado)
              FadeIn(
                child: Card(
                  elevation: 4,
                  child: Padding(
                    padding: const EdgeInsets.all(16.0),
                    child: Column(
                      children: [
                        Text(
                          '¡Bienvenido!',
                          style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
                        ),
                        SizedBox(height: 10),
                        Text(
                          'Ya estás verificado y puedes solicitar préstamos.',
                          style: TextStyle(fontSize: 16),
                        ),
                        SizedBox(height: 20),
                        DropdownButton<int>(
                          value: _plazoPrestamo,
                          items: [10, 15, 30].map((plazo) {
                            return DropdownMenuItem<int>(
                              value: plazo,
                              child: Text('Plazo: $plazo días'),
                            );
                          }).toList(),
                          onChanged: (value) {
                            setState(() {
                              _plazoPrestamo = value!;
                            });
                          },
                        ),
                        SizedBox(height: 20),
                        ScaleAnimation(
                          child: ElevatedButton(
                            onPressed: () => _solicitarPrestamo(200.0),
                            child: Text('Solicitar Préstamo'),
                          ),
                        ),
                        SizedBox(height: 20),
                        if (!_esSuscriptor)
                          ElevatedButton(
                            onPressed: _suscribirse,
                            child: Text('Suscribirse por Q50/mes'),
                          ),
                      ],
                    ),
                  ),
                ),
              ),
            SizedBox(height: 20),
            ElevatedButton(
              onPressed: () => Navigator.pushNamed(context, '/soporte'),
              child: Text('Contactar Soporte'),
            ),
          ],
        ),
      ),
    );
  }
}

class VerificacionScreen extends StatelessWidget {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  Future<void> _verificarDocumento(String docId) async {
    await _firestore.collection('verificaciones').doc(docId).update({
      'verificado': true,
    });
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Documento verificado exitosamente')),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Verificación de Documentos'),
      ),
      body: StreamBuilder<QuerySnapshot>(
        stream: _firestore
            .collection('verificaciones')
            .where('verificado', isEqualTo: false)
            .snapshots(),
        builder: (context, snapshot) {
          if (!snapshot.hasData) {
            return Center(child: CircularProgressIndicator());
          }
          final documentos = snapshot.data!.docs;
          return ListView.builder(
            itemCount: documentos.length,
            itemBuilder: (context, index) {
              final documento = documentos[index];
              return Card(
                margin: EdgeInsets.all(8.0),
                child: ListTile(
                  title: Text('Usuario: ${documento['usuarioId']}'),
                  subtitle: Text('Tipo: ${documento['tipo']}'),
                  trailing: IconButton(
                    icon: Icon(Icons.verified, color: Colors.green),
                    onPressed: () => _verificarDocumento(documento.id),
                  ),
                  onTap: () {
                    Navigator.push(
                      context,
                      MaterialPageRoute(
                        builder: (context) => DocumentoViewScreen(
                          url: documento['url'],
                        ),
                      ),
                    );
                  },
                ),
              );
            },
          );
        },
      ),
    );
  }
}

class DocumentoViewScreen extends StatelessWidget {
  final String url;

  const DocumentoViewScreen({required this.url});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Ver Documento'),
      ),
      body: Center(
        child: Hero(
          tag: url,
          child: Image.network(url),
        ),
      ),
    );
  }
}

class ConfiguracionScreen extends StatefulWidget {
  @override
  _ConfiguracionScreenState createState() => _ConfiguracionScreenState();
}

class _ConfiguracionScreenState extends State<ConfiguracionScreen> {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;
  final ImagePicker _picker = ImagePicker();
  File? _fotoPerfil;
  TextEditingController _telefonoController = TextEditingController();

  Future<void> _cambiarFotoPerfil() async {
    final XFile? image = await _picker.pickImage(source: ImageSource.gallery);
    if (image != null) {
      setState(() {
        _fotoPerfil = File(image.path);
      });
      final Reference ref = FirebaseStorage.instance
          .ref()
          .child('perfiles/${_auth.currentUser!.uid}');
      await ref.putFile(_fotoPerfil!);
      final String url = await ref.getDownloadURL();
      await _firestore.collection('usuarios').doc(_auth.currentUser!.uid).update({
        'fotoPerfil': url,
      });
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Foto de perfil actualizada')),
      );
    } else {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('No se seleccionó ninguna imagen')),
      );
    }
  }

  Future<void> _actualizarTelefono() async {
    if (_telefonoController.text.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Por favor, ingresa un número de teléfono')),
      );
      return;
    }
    await _firestore.collection('usuarios').doc(_auth.currentUser!.uid).update({
      'telefono': _telefonoController.text,
    });
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Número de teléfono actualizado')),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Configuración del Perfil'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            CircleAvatar(
              radius: 50,
              backgroundImage: _fotoPerfil != null
                  ? FileImage(_fotoPerfil!)
                  : AssetImage('assets/default_profile.png') as ImageProvider,
            ),
            SizedBox(height: 20),
            ElevatedButton(
              onPressed: _cambiarFotoPerfil,
              child: Text('Cambiar Foto de Perfil'),
            ),
            SizedBox(height: 20),
            TextField(
              controller: _telefonoController,
              decoration: InputDecoration(
                labelText: 'Número de Teléfono',
                border: OutlineInputBorder(),
              ),
            ),
            SizedBox(height: 20),
            ElevatedButton(
              onPressed: _actualizarTelefono,
              child: Text('Actualizar Teléfono'),
            ),
          ],
        ),
      ),
    );
  }
}

class SoporteScreen extends StatelessWidget {
  final String correoSoporte = 'soporte@savefood.store';
  final String numeroWhatsApp = '+50255786441';

  Future<void> _contactarWhatsApp() async {
    final String mensaje = 'Hola, quisiera hablar con un asesor para mi préstamo';
    final String url = 'https://wa.me/$numeroWhatsApp?text=${Uri.encodeFull(mensaje)}';
    if (await canLaunch(url)) {
      await launch(url);
    } else {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('No se pudo abrir WhatsApp')),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Soporte'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            Text(
              '¿Necesitas ayuda? Contáctanos:',
              style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 20),
            ListTile(
              leading: Icon(Icons.email),
              title: Text('Correo de Soporte'),
              subtitle: Text(correoSoporte),
              onTap: () async {
                final Uri emailUri = Uri(
                  scheme: 'mailto',
                  path: correoSoporte,
                );
                if (await canLaunch(emailUri.toString())) {
                  await launch(emailUri.toString());
                } else {
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('No se pudo abrir el correo')),
                  );
                }
              },
            ),
            ListTile(
              leading: Icon(Icons.phone),
              title: Text('WhatsApp de Soporte'),
              subtitle: Text(numeroWhatsApp),
              onTap: _contactarWhatsApp,
            ),
          ],
        ),
      ),
    );
  }
}

class FadeIn extends StatelessWidget {
  final Widget child;
  final Duration duration;

  const FadeIn({required this.child, this.duration = const Duration(milliseconds: 500)});

  @override
  Widget build(BuildContext context) {
    return TweenAnimationBuilder(
      tween: Tween<double>(begin: 0, end: 1),
      duration: duration,
      builder: (context, double value, child) {
        return Opacity(
          opacity: value,
          child: child,
        );
      },
      child: child,
    );
  }
}

class ScaleAnimation extends StatefulWidget {
  final Widget child;
  final Duration duration;

  const ScaleAnimation({required this.child, this.duration = const Duration(milliseconds: 200)});

  @override
  _ScaleAnimationState createState() => _ScaleAnimationState();
}

class _ScaleAnimationState extends State<ScaleAnimation> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: widget.duration,
    );
    _animation = Tween<double>(begin: 1, end: 0.95).animate(_controller);
  }

  void _onTapDown(TapDownDetails details) {
    _controller.forward();
  }

  void _onTapUp(TapUpDetails details) {
    _controller.reverse();
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTapDown: _onTapDown,
      onTapUp: _onTapUp,
      child: ScaleTransition(
        scale: _animation,
        child: widget.child,
      ),
    );
  }
}
