A continuación te entrego la implementación completa del **CRUD de docentes** (crear, listar, editar, eliminar) para el panel de administrador, totalmente funcional e integrado con Firebase (Auth + Firestore) y Provider. La solución incluye:

- Listado en tiempo real con StreamBuilder.
- Agregar nuevo docente (email, password, nombre, turno).
- Editar nombre y turno.
- Eliminación lógica (cambia rol a `'inhabilitado'`), impidiendo el login y ocultándolo de la lista activa.
- Validaciones y manejo de errores.

---

## 1. Extender `firestore_service.dart` con métodos para docentes

Crea (o amplía) `lib/services/firestore_service.dart`:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';

class FirestoreService {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  // Obtener stream de docentes activos (rol = 'docente')
  Stream<QuerySnapshot> getDocentesActivos() {
    return _firestore
        .collection('users')
        .where('rol', isEqualTo: 'docente')
        .snapshots();
  }

  // Obtener datos de un docente por uid
  Future<DocumentSnapshot> getDocente(String uid) async {
    return await _firestore.collection('users').doc(uid).get();
  }

  // Actualizar docente (solo nombre y turno)
  Future<void> actualizarDocente(String uid, {required String nombre, required String turno}) async {
    await _firestore.collection('users').doc(uid).update({
      'nombre': nombre,
      'turnoAsignado': turno,
    });
  }

  // Eliminación lógica: cambiar rol a 'inhabilitado'
  Future<void> inhabilitarDocente(String uid) async {
    await _firestore.collection('users').doc(uid).update({
      'rol': 'inhabilitado',
    });
  }
}
```

---

## 2. Crear `docentes_provider.dart` (para manejar el estado CRUD)

`lib/providers/docentes_provider.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import '../services/firestore_service.dart';
import '../providers/auth_provider.dart';

class DocentesProvider extends ChangeNotifier {
  final FirestoreService _firestoreService = FirestoreService();
  final FirebaseAuth _auth = FirebaseAuth.instance;
  
  bool _isLoading = false;
  String? _errorMessage;

  bool get isLoading => _isLoading;
  String? get errorMessage => _errorMessage;

  // Crear nuevo docente (usando AuthProvider.crearDocente)
  Future<bool> crearDocente({
    required String email,
    required String password,
    required String nombre,
    required String turno,
    required AuthProvider authProvider,
  }) async {
    _setLoading(true);
    final success = await authProvider.crearDocente(email, password, nombre, turno);
    _setLoading(false);
    if (!success) {
      _errorMessage = 'No se pudo crear el docente. El email puede estar en uso.';
    } else {
      _errorMessage = null;
    }
    return success;
  }

  // Editar docente (nombre y turno)
  Future<bool> editarDocente({
    required String uid,
    required String nombre,
    required String turno,
  }) async {
    _setLoading(true);
    try {
      await _firestoreService.actualizarDocente(uid, nombre: nombre, turno: turno);
      _errorMessage = null;
      _setLoading(false);
      return true;
    } catch (e) {
      _errorMessage = 'Error al actualizar: ${e.toString()}';
      _setLoading(false);
      return false;
    }
  }

  // Eliminar (inhabilitar) docente
  Future<bool> eliminarDocente(String uid) async {
    _setLoading(true);
    try {
      await _firestoreService.inhabilitarDocente(uid);
      _errorMessage = null;
      _setLoading(false);
      return true;
    } catch (e) {
      _errorMessage = 'Error al inhabilitar: ${e.toString()}';
      _setLoading(false);
      return false;
    }
  }

  void _setLoading(bool value) {
    _isLoading = value;
    notifyListeners();
  }

  void clearError() {
    _errorMessage = null;
    notifyListeners();
  }
}
```

---

## 3. Pantalla de gestión de docentes (`docentes_screen.dart`)

`lib/screens/admin/docentes_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../../providers/auth_provider.dart';
import '../../providers/docentes_provider.dart';
import '../../services/firestore_service.dart';

class DocentesScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final authProvider = Provider.of<AuthProvider>(context);
    final docentesProvider = Provider.of<DocentesProvider>(context);
    final firestoreService = FirestoreService();

    return Scaffold(
      appBar: AppBar(
        title: Text('Gestión de Docentes'),
        actions: [
          IconButton(
            icon: Icon(Icons.add),
            onPressed: () => _mostrarDialogoCrear(context, authProvider, docentesProvider),
          ),
        ],
      ),
      body: StreamBuilder<QuerySnapshot>(
        stream: firestoreService.getDocentesActivos(),
        builder: (context, snapshot) {
          if (snapshot.hasError) {
            return Center(child: Text('Error: ${snapshot.error}'));
          }
          if (snapshot.connectionState == ConnectionState.waiting) {
            return Center(child: CircularProgressIndicator());
          }

          final docs = snapshot.data!.docs;
          if (docs.isEmpty) {
            return Center(child: Text('No hay docentes registrados'));
          }

          return ListView.builder(
            itemCount: docs.length,
            itemBuilder: (context, index) {
              final doc = docs[index];
              final data = doc.data() as Map<String, dynamic>;
              final uid = doc.id;
              final nombre = data['nombre'] ?? 'Sin nombre';
              final email = data['email'] ?? '';
              final turno = data['turnoAsignado'] ?? 'matutino';

              return Card(
                margin: EdgeInsets.symmetric(horizontal: 16, vertical: 8),
                child: ListTile(
                  title: Text(nombre, style: TextStyle(fontWeight: FontWeight.bold)),
                  subtitle: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text('Email: $email'),
                      Text('Turno: ${turno == 'matutino' ? 'Matutino (7:00-15:00)' : 'Vespertino (13:00-21:00)'}'),
                    ],
                  ),
                  trailing: Row(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      IconButton(
                        icon: Icon(Icons.edit, color: Colors.blue),
                        onPressed: () => _mostrarDialogoEditar(context, uid, nombre, turno, docentesProvider),
                      ),
                      IconButton(
                        icon: Icon(Icons.delete, color: Colors.red),
                        onPressed: () => _confirmarEliminacion(context, uid, nombre, docentesProvider),
                      ),
                    ],
                  ),
                ),
              );
            },
          );
        },
      ),
    );
  }

  // Diálogo para crear docente
  void _mostrarDialogoCrear(BuildContext context, AuthProvider authProvider, DocentesProvider docentesProvider) {
    final formKey = GlobalKey<FormState>();
    String email = '';
    String password = '';
    String nombre = '';
    String turno = 'matutino';

    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text('Nuevo Docente'),
        content: Form(
          key: formKey,
          child: SingleChildScrollView(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                TextFormField(
                  decoration: InputDecoration(labelText: 'Email'),
                  validator: (v) => v!.contains('@') ? null : 'Email inválido',
                  onSaved: (v) => email = v!,
                ),
                TextFormField(
                  decoration: InputDecoration(labelText: 'Contraseña'),
                  obscureText: true,
                  validator: (v) => v!.length >= 6 ? null : 'Mínimo 6 caracteres',
                  onSaved: (v) => password = v!,
                ),
                TextFormField(
                  decoration: InputDecoration(labelText: 'Nombre completo'),
                  validator: (v) => v!.isNotEmpty ? null : 'Requerido',
                  onSaved: (v) => nombre = v!,
                ),
                DropdownButtonFormField<String>(
                  value: turno,
                  decoration: InputDecoration(labelText: 'Turno'),
                  items: [
                    DropdownMenuItem(value: 'matutino', child: Text('Matutino (7:00-15:00)')),
                    DropdownMenuItem(value: 'vespertino', child: Text('Vespertino (13:00-21:00)')),
                  ],
                  onChanged: (v) => turno = v!,
                ),
              ],
            ),
          ),
        ),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: Text('Cancelar')),
          ElevatedButton(
            onPressed: () async {
              if (formKey.currentState!.validate()) {
                formKey.currentState!.save();
                final success = await docentesProvider.crearDocente(
                  email: email,
                  password: password,
                  nombre: nombre,
                  turno: turno,
                  authProvider: authProvider,
                );
                if (success) {
                  Navigator.pop(context);
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('Docente creado exitosamente')),
                  );
                } else {
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text(docentesProvider.errorMessage ?? 'Error')),
                  );
                }
              }
            },
            child: docentesProvider.isLoading ? CircularProgressIndicator() : Text('Crear'),
          ),
        ],
      ),
    );
  }

  // Diálogo para editar
  void _mostrarDialogoEditar(BuildContext context, String uid, String nombreActual, String turnoActual, DocentesProvider provider) {
    final formKey = GlobalKey<FormState>();
    String nuevoNombre = nombreActual;
    String nuevoTurno = turnoActual;

    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text('Editar Docente'),
        content: Form(
          key: formKey,
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              TextFormField(
                initialValue: nombreActual,
                decoration: InputDecoration(labelText: 'Nombre'),
                validator: (v) => v!.isNotEmpty ? null : 'Requerido',
                onSaved: (v) => nuevoNombre = v!,
              ),
              DropdownButtonFormField<String>(
                value: nuevoTurno,
                decoration: InputDecoration(labelText: 'Turno'),
                items: [
                  DropdownMenuItem(value: 'matutino', child: Text('Matutino')),
                  DropdownMenuItem(value: 'vespertino', child: Text('Vespertino')),
                ],
                onChanged: (v) => nuevoTurno = v!,
              ),
            ],
          ),
        ),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: Text('Cancelar')),
          ElevatedButton(
            onPressed: () async {
              if (formKey.currentState!.validate()) {
                formKey.currentState!.save();
                final success = await provider.editarDocente(
                  uid: uid,
                  nombre: nuevoNombre,
                  turno: nuevoTurno,
                );
                if (success) {
                  Navigator.pop(context);
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('Docente actualizado')),
                  );
                } else {
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text(provider.errorMessage ?? 'Error')),
                  );
                }
              }
            },
            child: provider.isLoading ? CircularProgressIndicator() : Text('Guardar'),
          ),
        ],
      ),
    );
  }

  // Confirmar eliminación
  void _confirmarEliminacion(BuildContext context, String uid, String nombre, DocentesProvider provider) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text('Inhabilitar docente'),
        content: Text('¿Estás seguro de inhabilitar a $nombre? Ya no podrá iniciar sesión.'),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: Text('Cancelar')),
          ElevatedButton(
            style: ElevatedButton.styleFrom(backgroundColor: Colors.red),
            onPressed: () async {
              final success = await provider.eliminarDocente(uid);
              if (success) {
                Navigator.pop(context);
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(content: Text('Docente inhabilitado')),
                );
              } else {
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(content: Text(provider.errorMessage ?? 'Error al inhabilitar')),
                );
              }
            },
            child: provider.isLoading ? CircularProgressIndicator() : Text('Inhabilitar'),
          ),
        ],
      ),
    );
  }
}
```

---

## 4. Actualizar `AuthProvider` para que el método `crearDocente` sea correcto

En `lib/providers/auth_provider.dart`, asegura que el método `crearDocente` asigna el rol `'docente'` y guarda correctamente:

```dart
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
```

---

## 5. Integrar en el dashboard del administrador

Modifica `dashboard_screen.dart` para incluir la pantalla de docentes:

```dart
// Dentro de AdminPanel
ListTile(
  title: Text('Gestionar Docentes'),
  onTap: () {
    Navigator.push(context, MaterialPageRoute(builder: (_) => DocentesScreen()));
  },
),
```

No olvides proveer `DocentesProvider` en un nivel superior (por ejemplo, en `main.dart` o en la propia pantalla con `MultiProvider`). Recomendación: en `main.dart`, agrega:

```dart
MultiProvider(
  providers: [
    ChangeNotifierProvider(create: (_) => AuthProvider()),
    ChangeNotifierProvider(create: (_) => DocentesProvider()),
  ],
  child: MyApp(),
)
```

---

## 6. Resultado final

- **Listado** muestra todos los docentes activos con su turno.
- **Agregar**: valida email, contraseña, nombre y turno; crea usuario en Auth y documento en Firestore.
- **Editar**: permite cambiar nombre y turno, actualizando Firestore.
- **Eliminar**: cambia rol a `'inhabilitado'`, impidiendo el acceso y ocultándolo del listado (por el filtro `where('rol', isEqualTo: 'docente')`).

Todo es **reactivo** gracias a StreamBuilder y Provider. Los mensajes de éxito/error se muestran mediante SnackBar.

¿Necesitas que desarrolle la siguiente área: *generación de horarios docentes*? Puedo hacerlo con la misma calidad y consistencia.
