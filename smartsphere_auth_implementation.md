# SmartSphere Home System - Authentication Implementation

This document contains the complete code, schema, and instructions to implement the User Registration and Login system using Next.js (Frontend), Java Spring Boot (Backend), and Supabase (Database).

## 1. Supabase Database Schema

Run this SQL in your Supabase SQL Editor to create the users table:

```sql
CREATE TABLE users (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    user_id VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone_number VARCHAR(20) NOT NULL,
    address TEXT NOT NULL,
    password_hash VARCHAR(255), -- Nullable for Google OAuth users
    auth_provider VARCHAR(50) DEFAULT 'local', -- 'local' or 'google'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

---

## 2. Java Backend Implementation (Spring Boot)

Structure your `smartsphere-backend` project as follows:

### Application Properties (`src/main/resources/application.properties`)
```properties
spring.datasource.url=jdbc:postgresql://YOUR_SUPABASE_DB_URL:5432/postgres
spring.datasource.username=postgres
spring.datasource.password=YOUR_SUPABASE_PASSWORD
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# CORS config
frontend.url=https://YOUR_VERCEL_URL.vercel.app

# Google OAuth
spring.security.oauth2.client.registration.google.client-id=YOUR_GOOGLE_CLIENT_ID
spring.security.oauth2.client.registration.google.client-secret=YOUR_GOOGLE_CLIENT_SECRET
```

### pom.xml dependencies (Add these)
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-crypto</artifactId>
</dependency>
```

### Model (`src/main/java/com/assignment/smartsphere/model/User.java`)
```java
package com.assignment.smartsphere.model;

import jakarta.persistence.*;
import lombok.Data;
import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "users")
@Data
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private UUID id;

    @Column(unique = true, nullable = false)
    private String userId;

    @Column(nullable = false)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    private String phoneNumber;
    private String address;
    private String passwordHash;
    
    private String authProvider = "local";

    @Column(columnDefinition = "TIMESTAMP DEFAULT CURRENT_TIMESTAMP")
    private LocalDateTime createdAt = LocalDateTime.now();
}
```

### Repository (`src/main/java/com/assignment/smartsphere/repository/UserRepository.java`)
```java
package com.assignment.smartsphere.repository;

import com.assignment.smartsphere.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;
import java.util.UUID;

public interface UserRepository extends JpaRepository<User, UUID> {
    Optional<User> findByUserId(String userId);
    Optional<User> findByEmail(String email);
    boolean existsByUserId(String userId);
}
```

### Controller (`src/main/java/com/assignment/smartsphere/controller/AuthController.java`)
```java
package com.assignment.smartsphere.controller;

import com.assignment.smartsphere.model.User;
import com.assignment.smartsphere.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.web.bind.annotation.*;

import java.util.Map;
import java.util.Optional;

@RestController
@RequestMapping("/api/auth")
@CrossOrigin(origins = "${frontend.url}")
public class AuthController {

    @Autowired
    private UserRepository userRepository;
    private BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();

    @GetMapping("/check-userid")
    public ResponseEntity<?> checkUserId(@RequestParam String userId) {
        boolean exists = userRepository.existsByUserId(userId);
        return ResponseEntity.ok(Map.of("available", !exists));
    }

    @PostMapping("/register")
    public ResponseEntity<?> registerUser(@RequestBody User registrationData) {
        if (userRepository.existsByUserId(registrationData.getUserId())) {
            return ResponseEntity.badRequest().body("User ID already exists");
        }
        
        // Hash password
        String hashedPassword = passwordEncoder.encode(registrationData.getPasswordHash());
        registrationData.setPasswordHash(hashedPassword);
        
        User savedUser = userRepository.save(registrationData);
        return ResponseEntity.ok(Map.of("message", "Registration successful", "userId", savedUser.getUserId()));
    }

    @PostMapping("/login")
    public ResponseEntity<?> loginUser(@RequestBody Map<String, String> credentials) {
        String userId = credentials.get("userId");
        String password = credentials.get("password");

        Optional<User> userOpt = userRepository.findByUserId(userId);
        if (userOpt.isPresent() && passwordEncoder.matches(password, userOpt.get().getPasswordHash())) {
            return ResponseEntity.ok(Map.of("message", "Login successful", "userId", userId));
        }
        return ResponseEntity.status(401).body("Invalid credentials");
    }
}
```

### Security Configuration (`src/main/java/com/assignment/smartsphere/config/SecurityConfig.java`)
```java
package com.assignment.smartsphere.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        // Disable CSRF for REST APIs, allow public access to auth endpoints
        http.csrf().disable()
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .defaultSuccessUrl("https://YOUR_VERCEL_URL.vercel.app/dashboard", true)
            );
        return http.build();
    }
}
```

---

## 3. Frontend Implementation (Next.js / React)

Structure your `smartsphere-frontend` component. Use standard Tailwind standard colors.

### `app/register/page.jsx`

```jsx
"use client";
import { useState, useEffect } from 'react';
import { useRouter } from 'next/navigation';

export default function Register() {
  const router = useRouter();
  const [formData, setFormData] = useState({
    name: '', email: '', phone: '', address: '', userId: '', password: '', confirmPassword: ''
  });
  const [errors, setErrors] = useState({});
  const [isUserIdAvailable, setIsUserIdAvailable] = useState(null);
  
  // Replace with your Render deployed backend URL
  const API_URL = process.env.NEXT_PUBLIC_BACKEND_URL || 'http://localhost:8080/api/auth';

  const validateForm = () => {
    let newErrors = {};
    if (!formData.name) newErrors.name = 'Name is required';
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(formData.email)) newErrors.email = 'Valid email is required';
    if (!/^\d{10,}$/.test(formData.phone)) newErrors.phone = 'Valid phone number is required';
    if (!formData.address) newErrors.address = 'Address is required';
    if (!formData.userId) newErrors.userId = 'User ID is required';
    if (formData.password.length < 6) newErrors.password = 'Password must be at least 6 characters';
    if (formData.password !== formData.confirmPassword) newErrors.confirmPassword = 'Passwords do not match';
    
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const checkUserId = async (id) => {
    if (!id) return;
    try {
      const res = await fetch(`${API_URL}/check-userid?userId=${id}`);
      const data = await res.json();
      setIsUserIdAvailable(data.available);
      if (!data.available) setErrors(e => ({...e, userId: 'User ID is already taken'}));
    } catch (e) {
      console.error('Failed to check User ID');
    }
  };

  const handleChange = (e) => {
    setFormData({...formData, [e.target.name]: e.target.value});
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!validateForm() || isUserIdAvailable === false) return;

    try {
      const res = await fetch(`${API_URL}/register`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          name: formData.name, email: formData.email, phoneNumber: formData.phone, 
          address: formData.address, userId: formData.userId, passwordHash: formData.password
        })
      });

      if (res.ok) {
        router.push('/dashboard');
      } else {
        alert("Registration failed.");
      }
    } catch(err) {
      alert("Error connecting to server.");
    }
  };

  const handleGoogleLogin = () => {
    window.location.href = `${API_URL.replace('/api/auth', '')}/oauth2/authorization/google`;
  };

  const isFormValid = formData.name && formData.email && formData.phone && 
                      formData.address && formData.userId && formData.password && 
                      formData.password === formData.confirmPassword && isUserIdAvailable;

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-white flex items-center justify-center p-4">
      <div className="bg-white shadow-xl rounded-2xl p-8 max-w-md w-full border border-gray-100">
        <h2 className="text-3xl font-bold text-center text-blue-900 mb-6">SmartSphere</h2>
        <p className="text-center text-gray-500 mb-8">Create your smart home account</p>

        <form onSubmit={handleSubmit} className="space-y-4">
          <div>
            <input type="text" name="name" placeholder="Full Name" onChange={handleChange} className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-blue-500 outline-none" required />
            {errors.name && <p className="text-red-500 text-xs mt-1">{errors.name}</p>}
          </div>
          
          <div>
            <input type="email" name="email" placeholder="Email Address" onChange={handleChange} className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-blue-500 outline-none" required />
            {errors.email && <p className="text-red-500 text-xs mt-1">{errors.email}</p>}
          </div>

          <div>
            <input type="tel" name="phone" placeholder="Phone Number" onChange={handleChange} className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-blue-500 outline-none" required />
            {errors.phone && <p className="text-red-500 text-xs mt-1">{errors.phone}</p>}
          </div>

          <div>
            <input type="text" name="address" placeholder="Home Address" onChange={handleChange} className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-blue-500 outline-none" required />
            {errors.address && <p className="text-red-500 text-xs mt-1">{errors.address}</p>}
          </div>

          <div>
            <input type="text" name="userId" placeholder="Unique User ID" onBlur={(e) => checkUserId(e.target.value)} onChange={handleChange} className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-blue-500 outline-none" required />
            {isUserIdAvailable === true && <p className="text-green-500 text-xs mt-1">✓ User ID available</p>}
            {errors.userId && <p className="text-red-500 text-xs mt-1">{errors.userId}</p>}
          </div>

          <div>
            <input type="password" name="password" placeholder="Password" onChange={handleChange} className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-blue-500 outline-none" required />
            {errors.password && <p className="text-red-500 text-xs mt-1">{errors.password}</p>}
          </div>

          <div>
            <input type="password" name="confirmPassword" placeholder="Confirm Password" onChange={handleChange} className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-blue-500 outline-none" required />
            {errors.confirmPassword && <p className="text-red-500 text-xs mt-1">{errors.confirmPassword}</p>}
          </div>

          <button 
            type="submit" 
            disabled={!isFormValid}
            className={`w-full p-3 rounded-lg font-semibold text-white transition-all ${isFormValid ? 'bg-blue-600 hover:bg-blue-700' : 'bg-gray-300 cursor-not-allowed'}`}
          >
            Proceed Further
          </button>
        </form>

        <div className="mt-6 flex items-center justify-center space-x-2">
          <div className="h-px bg-gray-200 flex-1"></div>
          <span className="text-gray-400 text-sm">Or</span>
          <div className="h-px bg-gray-200 flex-1"></div>
        </div>

        <button 
          onClick={handleGoogleLogin} 
          className="mt-6 w-full p-3 border border-gray-300 rounded-lg flex items-center justify-center space-x-2 hover:bg-gray-50 transition-all"
        >
          <img src="https://www.svgrepo.com/show/475656/google-color.svg" alt="Google" className="w-5 h-5" />
          <span className="text-gray-700 font-medium">Sign in with Google</span>
        </button>
      </div>
    </div>
  );
}
```

---

## 4. Deployment Instructions

### Deploy Frontend to Vercel
1. In your frontend repository, add an environment variable `.env.local` containing:
   `NEXT_PUBLIC_BACKEND_URL=https://your-java-backend.onrender.com/api/auth`
2. Push your code to GitHub.
3. Import the repository in Vercel and deploy.

### Deploy Backend to Render
1. Ensure your `application.properties` does not hardcode passwords. Pass them via environment variables in Render:
   - `SPRING_DATASOURCE_URL`
   - `SPRING_DATASOURCE_PASSWORD`
   - `FRONTEND_URL=https://your-vercel-domain.vercel.app`
2. Connect your GitHub repository as a "Web Service" in Render.
3. Choose "Docker" or Java environment. Render will build and deploy correctly.

### Setup Google OAuth
1. Go to Google Cloud Console.
2. Create Credentials > OAuth Client ID.
3. Add Authorized Redirect URI: `https://your-java-backend.onrender.com/login/oauth2/code/google`
4. Copy Client ID and Secret and put them in Render Environment Variables.
