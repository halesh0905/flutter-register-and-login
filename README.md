Authentication System 🔐

A full-stack authentication application with login and registration functionality, built with:

Frontend: Flutter
Backend: Django + Django REST Framework
Database: Default Django DB (SQLite unless configured otherwise)
Authentication: Token-based (Django REST Framework Token Authentication)
📸 Screenshots

Place your <img width="1004" height="565" alt="Screenshot 2026-07-27 130121" src="https://github.com/user-attachments/assets/d4ae87c7-a375-41f1-b9a9-4fd18f680413" />
.png and <img width="1015" height="510" alt="Screenshot 2026-07-27 130525" src="https://github.com/user-attachments/assets/da575cb2-48ea-495c-a1fd-c32f90121261" />
.png screenshots in the same folder as this README so they render correctly.

✨ Features
User Registration: Create a new account with email and password
User Login: Authenticate existing users and receive auth token
Token Management: Secure token storage on the client side
Session Persistence: Remember logged-in user across app restarts
Logout: Clear authentication token and return to login screen
Input Validation: Email and password validation on both frontend and backend
Error Handling: User-friendly error messages for failed auth attempts
🗂️ Project Structure
.
├── flutter_app/
│   ├── lib/
│   │   ├── main.dart              # App entry point + navigation
│   │   ├── screens/
│   │   │   ├── login_screen.dart   # Login UI
│   │   │   ├── register_screen.dart# Registration UI
│   │   │   └── home_screen.dart    # Post-login home screen
│   │   ├── api_service.dart        # REST API client
│   │   └── models/
│   │       └── user.dart           # User data model
│   └── pubspec.yaml
│
└── django_backend/
    └── myapp/
        ├── models.py               # Custom User model (optional)
        ├── serializers.py          # Registration & Login serializers
        ├── views.py                # Register & Login ViewSets
        └── urls.py                 # Auth endpoint routes
    └── myproject/
        ├── settings.py             # INSTALLED_APPS config
        └── urls.py                 # Root URL config
🔧 Backend Setup (Django)
1. Create and activate a virtual environment
bash
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
2. Install dependencies
bash
pip install django djangorestframework
3. Add Django REST Framework to INSTALLED_APPS

In myproject/settings.py:

python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework.authtoken',  # Token authentication
    'myapp',
]

# Token Authentication Configuration
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
4. Apply migrations
bash
python manage.py makemigrations
python manage.py migrate
5. Create API endpoints

myapp/serializers.py:

python
from rest_framework import serializers
from django.contrib.auth.models import User

class UserRegistrationSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=6)
    password_confirm = serializers.CharField(write_only=True, min_length=6)

    class Meta:
        model = User
        fields = ['username', 'email', 'password', 'password_confirm']

    def validate(self, data):
        if data['password'] != data['password_confirm']:
            raise serializers.ValidationError({"password": "Passwords do not match."})
        return data

    def create(self, validated_data):
        validated_data.pop('password_confirm')
        user = User.objects.create_user(**validated_data)
        return user


class UserLoginSerializer(serializers.Serializer):
    username = serializers.CharField()
    password = serializers.CharField(write_only=True)

    def validate(self, data):
        from django.contrib.auth import authenticate
        user = authenticate(username=data['username'], password=data['password'])
        if not user:
            raise serializers.ValidationError("Invalid credentials.")
        data['user'] = user
        return data

myapp/views.py:

python
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.authtoken.models import Token
from django.contrib.auth.models import User
from .serializers import UserRegistrationSerializer, UserLoginSerializer

class AuthViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()

    @action(detail=False, methods=['post'])
    def register(self, request):
        serializer = UserRegistrationSerializer(data=request.data)
        if serializer.is_valid():
            user = serializer.save()
            token, _ = Token.objects.get_or_create(user=user)
            return Response({
                'message': 'User registered successfully.',
                'token': token.key,
                'username': user.username
            }, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    @action(detail=False, methods=['post'])
    def login(self, request):
        serializer = UserLoginSerializer(data=request.data)
        if serializer.is_valid():
            user = serializer.validated_data['user']
            token, _ = Token.objects.get_or_create(user=user)
            return Response({
                'message': 'Login successful.',
                'token': token.key,
                'username': user.username
            }, status=status.HTTP_200_OK)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

myapp/urls.py:

python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import AuthViewSet

router = DefaultRouter()
router.register(r'auth', AuthViewSet, basename='auth')

urlpatterns = [
    path('', include(router.urls)),
]

myproject/urls.py:

python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('myapp.urls')),
]
6. Run the server
bash
python manage.py runserver 0.0.0.0:8000
📡 API Endpoints
Method	Endpoint	Description	Request Body
POST	/api/auth/register/	Register a new user	{"username": "user", "email": "user@example.com", "password": "pass123", "password_confirm": "pass123"}
POST	/api/auth/login/	Authenticate user & get token	{"username": "user", "password": "pass123"}

Response Example (Login/Register Success):

json
{
    "message": "Login successful.",
    "token": "abc123def456...",
    "username": "john_doe"
}

Response Example (Error):

json
{
    "username": ["This field may not be blank."],
    "password": ["This field may not be blank."]
}
📱 Frontend Setup (Flutter)
1. Update pubspec.yaml
yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.0.0
  shared_preferences: ^2.0.0  # For storing auth token
2. Install dependencies
bash
flutter pub get
3. Create User Model

lib/models/user.dart:

dart
class User {
  final String username;
  final String email;
  final String token;

  User({
    required this.username,
    required this.email,
    required this.token,
  });

  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      username: json['username'] ?? '',
      email: json['email'] ?? '',
      token: json['token'] ?? '',
    );
  }
}
4. Create API Service

lib/api_service.dart:

dart
import 'package:http/http.dart' as http;
import 'package:shared_preferences/shared_preferences.dart';
import 'dart:convert';
import 'models/user.dart';

class ApiService {
  static const String baseUrl = 'http://127.0.0.1:8000/api';

  // For Android emulator: http://10.0.2.2:8000/api
  // For physical device: http://192.168.x.x:8000/api

  static Future<User?> register({
    required String username,
    required String email,
    required String password,
    required String passwordConfirm,
  }) async {
    try {
      final response = await http.post(
        Uri.parse('$baseUrl/auth/register/'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode({
          'username': username,
          'email': email,
          'password': password,
          'password_confirm': passwordConfirm,
        }),
      );

      if (response.statusCode == 201) {
        final data = jsonDecode(response.body);
        final user = User(
          username: data['username'],
          email: email,
          token: data['token'],
        );
        await _saveToken(data['token']);
        return user;
      } else {
        final errors = jsonDecode(response.body);
        throw Exception(errors.toString());
      }
    } catch (e) {
      throw Exception('Registration failed: $e');
    }
  }

  static Future<User?> login({
    required String username,
    required String password,
  }) async {
    try {
      final response = await http.post(
        Uri.parse('$baseUrl/auth/login/'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode({
          'username': username,
          'password': password,
        }),
      );

      if (response.statusCode == 200) {
        final data = jsonDecode(response.body);
        final user = User(
          username: data['username'],
          email: '',
          token: data['token'],
        );
        await _saveToken(data['token']);
        return user;
      } else {
        throw Exception('Invalid credentials');
      }
    } catch (e) {
      throw Exception('Login failed: $e');
    }
  }

  static Future<String?> getToken() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getString('auth_token');
  }

  static Future<void> _saveToken(String token) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('auth_token', token);
  }

  static Future<void> logout() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove('auth_token');
  }
}
5. Create Login Screen

lib/screens/login_screen.dart:

dart
import 'package:flutter/material.dart';
import '../api_service.dart';

class LoginScreen extends StatefulWidget {
  final Function(String) onLoginSuccess;

  const LoginScreen({Key? key, required this.onLoginSuccess}) : super(key: key);

  @override
  State<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _usernameController = TextEditingController();
  final _passwordController = TextEditingController();
  bool _isLoading = false;
  String? _errorMessage;

  void _login() async {
    setState(() => _isLoading = true);
    try {
      final user = await ApiService.login(
        username: _usernameController.text,
        password: _passwordController.text,
      );
      if (user != null) {
        widget.onLoginSuccess(user.token);
      }
    } catch (e) {
      setState(() => _errorMessage = e.toString());
    } finally {
      setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Login')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            TextField(
              controller: _usernameController,
              decoration: const InputDecoration(labelText: 'Username'),
            ),
            const SizedBox(height: 16),
            TextField(
              controller: _passwordController,
              decoration: const InputDecoration(labelText: 'Password'),
              obscureText: true,
            ),
            const SizedBox(height: 24),
            if (_errorMessage != null)
              Text(_errorMessage!, style: const TextStyle(color: Colors.red)),
            const SizedBox(height: 16),
            _isLoading
                ? const CircularProgressIndicator()
                : ElevatedButton(
                    onPressed: _login,
                    child: const Text('Login'),
                  ),
            const SizedBox(height: 16),
            TextButton(
              onPressed: () => Navigator.pushNamed(context, '/register'),
              child: const Text("Don't have an account? Register here"),
            ),
          ],
        ),
      ),
    );
  }

  @override
  void dispose() {
    _usernameController.dispose();
    _passwordController.dispose();
    super.dispose();
  }
}
6. Create Registration Screen

lib/screens/register_screen.dart:

dart
import 'package:flutter/material.dart';
import '../api_service.dart';

class RegisterScreen extends StatefulWidget {
  const RegisterScreen({Key? key}) : super(key: key);

  @override
  State<RegisterScreen> createState() => _RegisterScreenState();
}

class _RegisterScreenState extends State<RegisterScreen> {
  final _usernameController = TextEditingController();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  final _passwordConfirmController = TextEditingController();
  bool _isLoading = false;
  String? _errorMessage;

  void _register() async {
    setState(() => _isLoading = true);
    try {
      final user = await ApiService.register(
        username: _usernameController.text,
        email: _emailController.text,
        password: _passwordController.text,
        passwordConfirm: _passwordConfirmController.text,
      );
      if (user != null) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Registration successful! Logging in...')),
        );
        Navigator.pop(context);
      }
    } catch (e) {
      setState(() => _errorMessage = e.toString());
    } finally {
      setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Register')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: SingleChildScrollView(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              TextField(
                controller: _usernameController,
                decoration: const InputDecoration(labelText: 'Username'),
              ),
              const SizedBox(height: 16),
              TextField(
                controller: _emailController,
                decoration: const InputDecoration(labelText: 'Email'),
                keyboardType: TextInputType.emailAddress,
              ),
              const SizedBox(height: 16),
              TextField(
                controller: _passwordController,
                decoration: const InputDecoration(labelText: 'Password'),
                obscureText: true,
              ),
              const SizedBox(height: 16),
              TextField(
                controller: _passwordConfirmController,
                decoration: const InputDecoration(labelText: 'Confirm Password'),
                obscureText: true,
              ),
              const SizedBox(height: 24),
              if (_errorMessage != null)
                Text(_errorMessage!, style: const TextStyle(color: Colors.red)),
              const SizedBox(height: 16),
              _isLoading
                  ? const CircularProgressIndicator()
                  : ElevatedButton(
                      onPressed: _register,
                      child: const Text('Register'),
                    ),
              const SizedBox(height: 16),
              TextButton(
                onPressed: () => Navigator.pop(context),
                child: const Text('Already have an account? Login here'),
              ),
            ],
          ),
        ),
      ),
    );
  }

  @override
  void dispose() {
    _usernameController.dispose();
    _emailController.dispose();
    _passwordController.dispose();
    _passwordConfirmController.dispose();
    super.dispose();
  }
}
7. Update main.dart

lib/main.dart:

dart
import 'package:flutter/material.dart';
import 'screens/login_screen.dart';
import 'screens/register_screen.dart';
import 'screens/home_screen.dart';
import 'api_service.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final token = await ApiService.getToken();
  runApp(MyApp(isLoggedIn: token != null));
}

class MyApp extends StatefulWidget {
  final bool isLoggedIn;

  const MyApp({Key? key, required this.isLoggedIn}) : super(key: key);

  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  late bool _isLoggedIn;

  @override
  void initState() {
    super.initState();
    _isLoggedIn = widget.isLoggedIn;
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Authentication App',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: _isLoggedIn
          ? HomeScreen(onLogout: _handleLogout)
          : LoginScreen(onLoginSuccess: _handleLoginSuccess),
      routes: {
        '/register': (context) => const RegisterScreen(),
      },
    );
  }

  void _handleLoginSuccess(String token) {
    setState(() => _isLoggedIn = true);
  }

  void _handleLogout() {
    setState(() => _isLoggedIn = false);
  }
}
8. Create Home Screen

lib/screens/home_screen.dart:

dart
import 'package:flutter/material.dart';
import '../api_service.dart';

class HomeScreen extends StatelessWidget {
  final Function() onLogout;

  const HomeScreen({Key? key, required this.onLogout}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Home'),
        actions: [
          IconButton(
            icon: const Icon(Icons.logout),
            onPressed: () async {
              await ApiService.logout();
              onLogout();
            },
          ),
        ],
      ),
      body: const Center(
        child: Text('Welcome! You are logged in.'),
      ),
    );
  }
}
9. Run the app
bash
flutter run
🔐 Authentication Flow
Registration: User enters username, email, password → Sent to /api/auth/register/ → Token returned → Stored locally
Login: User enters username & password → Sent to /api/auth/login/ → Token returned → Stored locally
Session Persistence: App checks SharedPreferences for saved token on startup
Logout: Token removed from SharedPreferences → User redirected to login screen
🧱 Data Model
python
class User(models.Model):
    # Built-in Django User model (username, email, password)
    # Token created automatically via django.contrib.auth.models.User

class Token(models.Model):
    # Built-in Django REST Framework token model
    key = models.CharField(max_length=40, primary_key=True)
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    created = models.DateTimeField(auto_now_add=True)
🚀 How It Works
User fills in registration form (username, email, password)
Flutter sends POST request to /api/auth/register/
Django validates input and creates User + Token
Token returned to Flutter and saved in SharedPreferences
App navigates to home screen
On next app launch, app checks for saved token and skips login if found
User can logout to clear token and return to login screen
📝 Notes & Possible Improvements
✅ Add email verification (send confirmation email on registration)
✅ Add password reset functionality
✅ Add refresh token mechanism for better security
✅ Implement input validation (email format, password strength) on both frontend and backend
✅ Add loading indicators and error handling for all network calls
✅ Add CORS configuration (django-cors-headers) if frontend runs on different host
✅ Use HTTPS in production (not HTTP)
✅ Add social login (Google, GitHub, etc.)
✅ Implement remember me functionality
✅ Add biometric authentication (fingerprint/face ID on mobile)
📄 License

This project is provided as-is for learning/demo purposes.
