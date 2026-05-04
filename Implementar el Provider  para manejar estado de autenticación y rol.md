Claro, a continuación te presento la implementación completa de **Provider** para manejar el estado de autenticación y el rol del usuario en la aplicación de **Control de Asistencia**. Este código se integra con Firebase Auth y Firestore y permite:

- Iniciar sesión con email/contraseña.
- Registrar nuevos usuarios (solo admin puede crear docentes, pero aquí muestro cómo crear el primer admin manualmente o desde registro).
- Escuchar cambios en el estado de autenticación.
- Obtener el rol del usuario desde Firestore.
- Proveer el estado a toda la app, con redirección automática a la pantalla adecuada (login, dashboard de admin o docente).

---

## 1. Agregar dependencias en `pubspec.yaml`

```yaml
dependencies:
  flutter:
    sdk: flutter
  provider: ^6.1.2
  firebase_core: ^3.1.1
  firebase_auth: ^5.1.4
  cloud_firestore: ^5.2.0
  # ... otras dependencias (local_auth, etc.)
```

Luego ejecuta `flutter pub get`.

---

## 2. Crear el proveedor `AuthProvider`

Crea `lib/providers/auth_provider.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

class AuthProvider extends ChangeNotifier {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  User? _user;
  String? _rol;
  bool _isLoading = false;

  User? get user => _user;
  String? get rol => _rol;
  bool get isLoading => _isLoading;
  bool get isAdmin => _rol == 'admin';
  bool get isDocente => _rol == 'docente';
  bool get isAuthenticated => _user != null;

  AuthProvider() {
    _init();
  }

  // Escuchar cambios de autenticación
  void _init() {
    _auth.authStateChanges().listen((User? user) async {
      _user = user;
      if (user != null) {
        await _cargarRol(user.uid);
      } else {
        _rol = null;
      }
      notifyListeners();
    });
  }

  // Cargar el rol desde Firestore
  Future<void> _cargarRol(String uid) async {
    try {
      final doc = await _firestore.collection('users').doc(uid).get();
      if (doc.exists) {
        _rol = doc.data()?['rol'] as String?;
      } else {
        _rol = null;
      }
    } catch (e) {
      debugPrint('Error cargando rol: $e');
      _rol = null;
    }
    notifyListeners();
  }

  // Iniciar sesión
  Future<bool> login(String email, String password) async {
    _setLoading(true);
    try {
      await _auth.signInWithEmailAndPassword(email: email, password: password);
      return true;
    } on FirebaseAuthException catch (e) {
      _setLoading(false);
      _mostrarError(e.message ?? 'Error de autenticación');
      return false;
    } catch (e) {
      _setLoading(false);
      _mostrarError('Ocurrió un error inesperado');
      return false;
    }
  }

  // Registrar un nuevo usuario (solo debería usarse para crear el admin inicial)
  // En producción, el registro de docentes lo hará el admin desde el panel.
  Future<bool> register(String email, String password, String nombre, String rol) async {
    _setLoading(true);
    try {
      UserCredential cred = await _auth.createUserWithEmailAndPassword(
        email: email,
        password: password,
      );
      // Guardar en Firestore
      await _firestore.collection('users').doc(cred.user!.uid).set({
        'email': email,
        'nombre': nombre,
        'rol': rol,
        'turnoAsignado': 'matutino', // valor por defecto
        'huellaRegistrada': false,
      });
      return true;
    } on FirebaseAuthException catch (e) {
      _setLoading(false);
      _mostrarError(e.message ?? 'Error en registro');
      return false;
    } catch (e) {
      _setLoading(false);
      _mostrarError('Error inesperado');
      return false;
    }
  }

  // Cerrar sesión
  Future<void> logout() async {
    await _auth.signOut();
    _user = null;
    _rol = null;
    notifyListeners();
  }

  // Método para que el admin cree un nuevo docente (ya autenticado como admin)
  Future<bool> crearDocente(String email, String password, String nombre, String turno) async {
    if (!isAdmin) {
      _mostrarError('Solo administradores pueden crear docentes');
      return false;
    }
    _setLoading(true);
    try {
      UserCredential cred = await _auth.createUserWithEmailAndPassword(
        email: email,
        password: password,
      );
      await _firestore.collection('users').doc(cred.user!.uid).set({
        'email': email,
        'nombre': nombre,
        'rol': 'docente',
        'turnoAsignado': turno,
        'huellaRegistrada': false,
      });
      return true;
    } on FirebaseAuthException catch (e) {
      _setLoading(false);
      _mostrarError(e.message ?? 'Error al crear docente');
      return false;
    } catch (e) {
      _setLoading(false);
      _mostrarError('Error inesperado');
      return false;
    }
  }

  void _setLoading(bool value) {
    _isLoading = value;
    notifyListeners();
  }

  void _mostrarError(String mensaje) {
    // Puedes usar un SnackBar o un diálogo global. Aquí lanzamos una excepción o manejas con un callback.
    debugPrint('Error: $mensaje');
    // Opcional: usar un scaffold messenger key global.
  }
}
```

---

## 3. Integrar el Provider en `main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:firebase_core/firebase_core.dart';
import 'providers/auth_provider.dart';
import 'screens/login_screen.dart';
import 'screens/dashboard_screen.dart';
import 'screens/loading_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (_) => AuthProvider(),
      child: MaterialApp(
        title: 'Control de Asistencia',
        theme: ThemeData(
          primarySwatch: Colors.blue,
          visualDensity: VisualDensity.adaptivePlatformDensity,
        ),
        home: Consumer<AuthProvider>(
          builder: (context, authProvider, _) {
            if (authProvider.isLoading) {
              return LoadingScreen(); // Pantalla de carga
            }
            if (authProvider.isAuthenticated) {
              // Redirigir según el rol
              return DashboardScreen();
            } else {
              return LoginScreen();
            }
          },
        ),
      ),
    );
  }
}
```

---

## 4. Uso en la pantalla de Login

`lib/screens/login_screen.dart` (extracto):

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../providers/auth_provider.dart';

class LoginScreen extends StatelessWidget {
  final TextEditingController emailController = TextEditingController();
  final TextEditingController passController = TextEditingController();

  @override
  Widget build(BuildContext context) {
    final authProvider = Provider.of<AuthProvider>(context);

    return Scaffold(
      body: Center(
        child: Padding(
          padding: EdgeInsets.all(24),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              TextField(
                controller: emailController,
                decoration: InputDecoration(labelText: 'Correo'),
              ),
              TextField(
                controller: passController,
                obscureText: true,
                decoration: InputDecoration(labelText: 'Contraseña'),
              ),
              SizedBox(height: 20),
              ElevatedButton(
                onPressed: authProvider.isLoading
                    ? null
                    : () async {
                        bool success = await authProvider.login(
                          emailController.text,
                          passController.text,
                        );
                        if (!success) {
                          ScaffoldMessenger.of(context).showSnackBar(
                            SnackBar(content: Text('Error al iniciar sesión')),
                          );
                        }
                      },
                child: authProvider.isLoading
                    ? CircularProgressIndicator()
                    : Text('Iniciar sesión'),
              ),
              TextButton(
                onPressed: () {
                  // Navegar a registro (solo para el admin inicial, luego ocultar)
                  Navigator.pushNamed(context, '/register');
                },
                child: Text('Registrarse (admin inicial)'),
              )
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## 5. Uso en el Dashboard (según rol)

`lib/screens/dashboard_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../providers/auth_provider.dart';

class DashboardScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final authProvider = Provider.of<AuthProvider>(context);
    final rol = authProvider.rol;

    return Scaffold(
      appBar: AppBar(
        title: Text('Dashboard - ${rol?.toUpperCase()}'),
        actions: [
          IconButton(
            icon: Icon(Icons.logout),
            onPressed: () => authProvider.logout(),
          ),
        ],
      ),
      body: Center(
        child: rol == 'admin'
            ? AdminPanel()
            : DocentePanel(),
      ),
    );
  }
}

class AdminPanel extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ListView(
      children: [
        ListTile(title: Text('Gestionar Docentes'), onTap: () {}),
        ListTile(title: Text('Generar Horarios'), onTap: () {}),
        ListTile(title: Text('Reportes Globales'), onTap: () {}),
        ListTile(title: Text('Marcar Asistencia (Prueba)'), onTap: () {}),
      ],
    );
  }
}

class DocentePanel extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ListView(
      children: [
        ListTile(title: Text('Mi Horario'), onTap: () {}),
        ListTile(title: Text('Marcar Asistencia'), onTap: () {}),
        ListTile(title: Text('Mis Reportes'), onTap: () {}),
        ListTile(title: Text('Consultas'), onTap: () {}),
      ],
    );
  }
}
```

---

## 6. Consideraciones adicionales

- **Protección de rutas**: En cada pantalla sensible (ej. crear docente), puedes verificar `authProvider.isAdmin` y mostrar un error o redirigir.
- **Carga del rol**: El provider carga el rol durante el `authStateChanges`. Si el usuario existe en Firebase Auth pero no tiene documento en Firestore, `_rol` será `null`. Maneja ese caso redirigiendo a una pantalla de error o completando perfil.
- **Registro inicial**: Para crear el primer administrador, puedes crear un usuario manualmente en Firebase Console o usar un script. El método `register` del provider asigna el rol que le pases. Asegúrate de que solo se pueda llamar una vez o bajo condiciones especiales.
- **Optimización**: Usa `Consumer` en lugar de `Provider.of` cuando solo necesites escuchar cambios en partes específicas del árbol para evitar reconstrucciones innecesarias.

Con este `AuthProvider` ya tienes una base sólida para manejar autenticación y roles en toda la app, con redirección automática y carga de datos desde Firestore. ¿Necesitas que desarrolle alguna de las pantallas específicas (gestión de docentes, generación de horarios, etc.) con esta misma arquitectura?
