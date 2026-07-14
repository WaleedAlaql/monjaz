Monjaz — Task and Habit Tracker

A full-stack mobile application for managing daily tasks and building habits, with real-time notifications delivered via WebSocket the moment a deadline approaches.


Overview

Monjaz is built as two independent projects: a Spring Boot REST API handling all business logic, and a Flutter mobile client consuming it. The two communicate over HTTP for CRUD operations and over a persistent WebSocket connection for push-style notifications without polling.


Architecture

Flutter (iOS / Android)
    |
    |-- HTTP (REST API) --------> Spring Boot
    |                                  |-- PostgreSQL  (primary data store)
    |                                  |-- Redis       (response caching)
    |
    |-- WebSocket (STOMP) ------> Spring Boot
                                       |-- Scheduler  (fires every 60 s)
                                       |-- Notification dispatch


Features

Task management


Create, update, and delete tasks with title, description, due date/time, and priority level
Mark tasks as complete
Automatic status transitions: Pending -> In Progress -> Completed / Overdue
Tasks sorted by due date ascending


Habit tracking


Create daily or weekly habits with a reminder time
Log completions with one tap
Streak tracking: current streak and all-time longest streak
Soft delete (archive) so historical logs are preserved


Real-time notifications


A background scheduler runs every 60 seconds
Tasks due within the next 15 minutes trigger a "due soon" alert
Tasks whose deadline has passed trigger an "overdue" alert
Alerts are delivered immediately to the connected device via WebSocket, with no polling required
On iOS, notifications appear as native system alerts even when the app is in the background


Security


Stateless JWT authentication on every API request
WebSocket connections are authenticated by reading the Bearer token from the STOMP CONNECT frame, so the broker can route notifications to the correct user



Tech Stack

LayerTechnologyBackend languageJava 17Backend frameworkSpring Boot 3DatabasePostgreSQLCachingRedis (Spring Cache with 10-minute TTL)Real-timeWebSocket + STOMPAuthenticationSpring Security + JWT (jjwt)SchedulingSpring @ScheduledFrontend languageDartFrontend frameworkFlutterState managementRiverpod (with code generation)HTTP clientdart:httpLocal notificationsflutter_local_notificationsToken storageSharedPreferences


Project Structure

monjaz/
|-- monjaz-backend/
|   |-- src/main/java/com/monjaz/
|   |   |-- config/          SecurityConfig, WebSocketConfig, RedisConfig, WebSocketAuthInterceptor
|   |   |-- controller/      AuthController, TaskController, HabitController
|   |   |-- dto/             request/ and response/ records
|   |   |-- entity/          User, Task, Habit, HabitLog
|   |   |-- exception/       GlobalExceptionHandler and custom exception types
|   |   |-- repository/      JPA repositories with derived query methods
|   |   |-- security/        JwtUtil, JwtAuthFilter, UserDetailsServiceImpl
|   |   |-- service/         AuthService, TaskService, HabitService,
|   |                        NotificationService, ScheduledNotificationService
|   |-- src/main/resources/
|       |-- application.properties
|
|-- monjaz_flutter/
    |-- lib/
        |-- app/             App entry point and HomeScreen
        |-- core/            ApiConstants, HttpClient, StorageService,
        |                    LocalNotificationService, ApiResult
        |-- features/
            |-- auth/        model, service, provider, view
            |-- tasks/       model, service, provider, view
            |-- habits/      model, service, provider, view
            |-- notifications/ model, service
        |-- shared/
            |-- widgets/     TaskCard, HabitCard


API Reference

Authentication

MethodEndpointDescriptionPOST/api/auth/registerRegister a new userPOST/api/auth/loginLogin and receive a JWT

Tasks

MethodEndpointDescriptionGET/api/tasksGet all tasks for the authenticated userPOST/api/tasksCreate a taskGET/api/tasks/{id}Get a single taskPUT/api/tasks/{id}Update a taskPATCH/api/tasks/{id}/completeMark a task as completeDELETE/api/tasks/{id}Delete a task

Habits

MethodEndpointDescriptionGET/api/habitsGet all habits for the authenticated userPOST/api/habitsCreate a habitPATCH/api/habits/{id}/completeLog today's completionDELETE/api/habits/{id}Archive a habit

WebSocket

Connect to ws://{host}/ws with the STOMP header Authorization: Bearer {token}.

Subscribe to /user/queue/notifications to receive real-time alerts.


Getting Started

Prerequisites


Java 17
Maven
PostgreSQL 14+
Redis
Flutter 3.x


Backend setup

bash# 1. Create the database
psql postgres -c "CREATE DATABASE monjaz_db;"

# 2. Create a .env file in monjaz-backend/
DB_USERNAME=postgres
DB_PASSWORD=your_password
REDIS_HOST=localhost
REDIS_PORT=6379
JWT_SECRET=your_long_random_secret
JWT_EXPIRATION_MS=86400000

# 3. Run
cd monjaz-backend
mvn spring-boot:run

The API starts on port 8000 by default.

Flutter setup

bashcd monjaz_flutter
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter run

Update lib/core/constants/api_constants.dart with the correct host:


iOS Simulator: localhost
Android Emulator: 10.0.2.2
Physical device: the machine's local IP address



How the Notification Pipeline Works


The Flutter client connects to the WebSocket endpoint after login and subscribes to /user/queue/notifications.
WebSocketAuthInterceptor reads the JWT from the STOMP CONNECT frame and registers the user identity with the broker.
ScheduledNotificationService runs two jobs every 60 seconds:

Query for tasks due within the next 15 minutes where reminder_sent = false. For each match, send a notification via SimpMessagingTemplate.convertAndSendToUser and flip reminder_sent = true to prevent repeat alerts.
Query for tasks past their due date that are not yet marked overdue. Update their status and send an overdue alert.



The Flutter client receives the STOMP frame, deserializes the payload, and calls flutter_local_notifications to display a native system notification.



Caching Strategy

Task and habit lists are cached in Redis per user with a 10-minute TTL. Any write operation (create, update, complete, delete) evicts the relevant cache entry so the next read fetches fresh data from PostgreSQL. This is implemented with Spring's @Cacheable and @CacheEvict annotations, keeping the service layer free of manual cache management.


Author

Waleed Aql Alaql — LinkedIn
