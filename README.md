Okay, let's integrate your PHP pages with the provided database schema, make the icons functional, handle dynamic data, implement security measures, and improve visual consistency.

We will create a simple `includes/config.php` file to handle common things like starting the session and including the database connection. We'll also add necessary columns to the `users` table in the SQL schema provided.

**Database Schema Update:**

Add the `full_name`, `age`, and `gender` columns to the `users` table schema.

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS ratingtales;
USE ratingtales;

-- Users table
CREATE TABLE users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL, -- Added
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    age INT, -- Added
    gender ENUM('Laki-laki', 'Perempuan'), -- Added
    profile_image VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Movies table
CREATE TABLE movies (
    movie_id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    summary TEXT,
    release_date DATE NOT NULL,
    duration_hours INT NOT NULL,
    duration_minutes INT NOT NULL,
    age_rating ENUM('G', 'PG', 'PG-13', 'R', 'NC-17') NOT NULL,
    poster_image VARCHAR(255), -- Made nullable for simplicity in upload example
    trailer_url VARCHAR(255),
    trailer_file VARCHAR(255),
    uploaded_by INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (uploaded_by) REFERENCES users(user_id) ON DELETE CASCADE -- Added CASCADE for uploaded movies
);

-- Movie genres table
CREATE TABLE movie_genres (
    movie_id INT NOT NULL,
    genre ENUM('action', 'adventure', 'comedy', 'drama', 'horror', 'supernatural', 'animation', 'sci-fi') NOT NULL, -- Added more genres from examples
    PRIMARY KEY (movie_id, genre),
    FOREIGN KEY (movie_id) REFERENCES movies(movie_id) ON DELETE CASCADE
);

-- Reviews table
CREATE TABLE reviews (
    review_id INT AUTO_INCREMENT PRIMARY KEY,
    movie_id INT NOT NULL,
    user_id INT NOT NULL,
    rating DECIMAL(2,1) NOT NULL, -- Example: 4.5
    comment TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (movie_id) REFERENCES movies(movie_id) ON DELETE CASCADE, -- Added CASCADE
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE -- Added CASCADE
);

-- Favorites table
CREATE TABLE favorites (
    favorite_id INT AUTO_INCREMENT PRIMARY KEY,
    movie_id INT NOT NULL,
    user_id INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (movie_id) REFERENCES movies(movie_id) ON DELETE CASCADE, -- Added CASCADE
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE, -- Added CASCADE
    UNIQUE KEY unique_favorite (movie_id, user_id)
);

-- Create indexes
CREATE INDEX idx_movie_title ON movies(title);
CREATE INDEX idx_movie_genre ON movie_genres(genre);
CREATE INDEX idx_movie_rating ON reviews(movie_id, rating); -- Not strictly needed for average, but good
CREATE INDEX idx_user_favorites ON favorites(user_id, created_at);
CREATE INDEX idx_movie_uploaded_by ON movies(uploaded_by); -- Added for profile/manage page
CREATE INDEX idx_review_movie_user ON reviews(movie_id, user_id); -- Added for user reviews

```

**1. `includes/config.php` (New File)**

This file will handle session starting, including the database connection, defining helper functions, and handling authentication checks.

```php
<?php
// includes/config.php

// Start a session if one hasn't been started already
if (session_status() == PHP_SESSION_NONE) {
    session_start();
}

// Include the database connection file
require_once __DIR__ . '/../config/database.php'; // Use __DIR__ for reliable path

// Helper function to generate random string for CAPTCHA
function generateRandomString($length = 6) {
    $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    $charactersLength = strlen($characters);
    $randomString = '';
    for ($i = 0; $i < $length; $i++) {
        $randomString .= $characters[rand(0, $charactersLength - 1)];
    }
    return $randomString;
}

// Function to check if a user is authenticated
function isAuthenticated() {
    return isset($_SESSION['user_id']) && $_SESSION['user_id'] > 0;
}

// Function to redirect to login if not authenticated
function redirectIfNotAuthenticated() {
    if (!isAuthenticated()) {
        // Store the intended URL before redirecting
        $_SESSION['intended_url'] = $_SERVER['REQUEST_URI'];
        header('Location: ../autentikasi/form-login.php');
        exit;
    }
}

// Function to fetch user details by ID
// Assuming getUserById function exists in config/database.php
function getAuthenticatedUser() {
    if (isAuthenticated()) {
        global $pdo;
        $userId = $_SESSION['user_id'];
        // Ensure the user still exists in the database
        $stmt = $pdo->prepare("SELECT user_id, full_name, username, email, profile_image, age, gender, bio FROM users WHERE user_id = ?");
        $stmt->execute([$userId]);
        $user = $stmt->fetch();
        if ($user) {
            // Store in session or return, depending on need
            // For now, just return
            return $user;
        } else {
            // User not found in DB (maybe deleted?), clear session
            session_destroy();
            return null;
        }
    }
    return null;
}

// Function to calculate average rating for a movie
function getMovieAverageRating($movieId) {
    global $pdo;
    $stmt = $pdo->prepare("SELECT AVG(rating) as average_rating FROM reviews WHERE movie_id = ?");
    $stmt->execute([$movieId]);
    $result = $stmt->fetch();
    // Return formatted rating or 'N/A'
    return $result && $result['average_rating'] !== null ? number_format($result['average_rating'], 1) : 'N/A';
}

// Function to check if a movie is favorited by the current user
function isMovieFavorited($movieId, $userId) {
    global $pdo;
    if (!$userId) return false; // Cannot favorite if not logged in
    $stmt = $pdo->prepare("SELECT COUNT(*) FROM favorites WHERE movie_id = ? AND user_id = ?");
    $stmt->execute([$movieId, $userId]);
    return $stmt->fetchColumn() > 0;
}

// Define upload directories
define('UPLOAD_DIR_POSTERS', '../uploads/posters/');
define('UPLOAD_DIR_TRAILERS', '../uploads/trailers/');

// Create upload directories if they don't exist
if (!is_dir(UPLOAD_DIR_POSTERS)) {
    mkdir(UPLOAD_DIR_POSTERS, 0777, true);
}
if (!is_dir(UPLOAD_DIR_TRAILERS)) {
    mkdir(UPLOAD_DIR_TRAILERS, 0777, true);
}
?>
```

**2. `config/database.php` (Modified)**

Update existing functions to use prepared statements consistently and add the new functions required.

```php
<?php
// config/database.php

$host = 'localhost';
$dbname = 'ratingtales';
$username = 'root';
$password = '';

try {
    $pdo = new PDO("mysql:host=$host;dbname=$dbname", $username, $password);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    $pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
} catch(PDOException $e) {
    // Log the error instead of dying on a live site
    error_log("Database connection failed: " . $e->getMessage());
    // Provide a user-friendly message
    die("Oops! Something went wrong with the database connection. Please try again later.");
}

// User Functions
// Added full_name, age, gender to createUser
function createUser($full_name, $username, $email, $password, $age, $gender, $profile_image = null) {
    global $pdo;
    $hashedPassword = password_hash($password, PASSWORD_BCRYPT); // Use BCRYPT for better security
    $stmt = $pdo->prepare("INSERT INTO users (full_name, username, email, password, age, gender, profile_image) VALUES (?, ?, ?, ?, ?, ?, ?)");
    return $stmt->execute([$full_name, $username, $email, $hashedPassword, $age, $gender, $profile_image]);
}

function getUserById($userId) {
    global $pdo;
    $stmt = $pdo->prepare("SELECT user_id, full_name, username, email, profile_image, age, gender, bio FROM users WHERE user_id = ?");
    $stmt->execute([$userId]);
    return $stmt->fetch();
}

// Added updateUser function
function updateUser($userId, $data) {
    global $pdo;
    $updates = [];
    $params = [];
    foreach ($data as $key => $value) {
        // Basic validation for allowed update fields
        if (in_array($key, ['full_name', 'username', 'email', 'profile_image', 'age', 'gender', 'bio'])) {
            $updates[] = "`{$key}` = ?";
            $params[] = $value;
        }
    }

    if (empty($updates)) {
        return false; // Nothing to update
    }

    $sql = "UPDATE users SET " . implode(', ', $updates) . " WHERE user_id = ?";
    $params[] = $userId;

    $stmt = $pdo->prepare($sql);
    return $stmt->execute($params);
}


// Movie Functions
// Added uploaded_by
function createMovie($title, $summary, $release_date, $duration_hours, $duration_minutes, $age_rating, $poster_image, $trailer_url, $trailer_file, $uploaded_by) {
    global $pdo;
    $stmt = $pdo->prepare("INSERT INTO movies (title, summary, release_date, duration_hours, duration_minutes, age_rating, poster_image, trailer_url, trailer_file, uploaded_by) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)");
    $stmt->execute([$title, $summary, $release_date, $duration_hours, $duration_minutes, $age_rating, $poster_image, $trailer_url, $trailer_file, $uploaded_by]);
    return $pdo->lastInsertId(); // Return the ID of the newly created movie
}

function addMovieGenre($movie_id, $genre) {
    global $pdo;
    $stmt = $pdo->prepare("INSERT INTO movie_genres (movie_id, genre) VALUES (?, ?)");
    // Ignore duplicate genre insertion errors if a genre is submitted twice for the same movie
    return $stmt->execute([$movie_id, $genre]);
}

function getMovieById($movieId) {
    global $pdo;
    // Fetch movie details along with its genres and the uploader's username
    $stmt = $pdo->prepare("
        SELECT
            m.*,
            GROUP_CONCAT(mg.genre) as genres,
            u.username as uploader_username
        FROM movies m
        LEFT JOIN movie_genres mg ON m.movie_id = mg.movie_id
        JOIN users u ON m.uploaded_by = u.user_id
        WHERE m.movie_id = ?
        GROUP BY m.movie_id
    ");
    $stmt->execute([$movieId]);
    $movie = $stmt->fetch();

    // Add average rating to the movie data
    if ($movie) {
        $movie['average_rating'] = getMovieAverageRating($movieId); // Use the helper function
    }

    return $movie;
}


function getAllMovies() {
    global $pdo;
    // Fetch all movies along with genres and average rating
    $stmt = $pdo->prepare("
        SELECT
            m.*,
            GROUP_CONCAT(mg.genre) as genres,
            (SELECT AVG(rating) FROM reviews WHERE movie_id = m.movie_id) as average_rating
        FROM movies m
        LEFT JOIN movie_genres mg ON m.movie_id = mg.movie_id
        GROUP BY m.movie_id
        ORDER BY m.created_at DESC
    ");
    $stmt->execute();
    return $stmt->fetchAll();
}

// Added function to get movies uploaded by a specific user
function getUserUploadedMovies($userId) {
    global $pdo;
    $stmt = $pdo->prepare("
        SELECT
            m.*,
            GROUP_CONCAT(mg.genre) as genres,
            (SELECT AVG(rating) FROM reviews WHERE movie_id = m.movie_id) as average_rating
        FROM movies m
        LEFT JOIN movie_genres mg ON m.movie_id = mg.movie_id
        WHERE m.uploaded_by = ?
        GROUP BY m.movie_id
        ORDER BY m.created_at DESC
    ");
    $stmt->execute([$userId]);
    return $stmt->fetchAll();
}


// Review Functions
function createReview($movie_id, $user_id, $rating, $comment) {
    global $pdo;
    // Prevent duplicate reviews by the same user for the same movie
    // Or update existing review? Let's upsert (insert or update)
    $stmt = $pdo->prepare("
        INSERT INTO reviews (movie_id, user_id, rating, comment)
        VALUES (?, ?, ?, ?)
        ON DUPLICATE KEY UPDATE rating = VALUES(rating), comment = VALUES(comment)
    ");
    return $stmt->execute([$movie_id, $user_id, $rating, $comment]);
}

function getMovieReviews($movieId) {
    global $pdo;
    // Fetch reviews along with reviewer's username and profile image
    $stmt = $pdo->prepare("
        SELECT
            r.*,
            u.username,
            u.profile_image
        FROM reviews r
        JOIN users u ON r.user_id = u.user_id
        WHERE r.movie_id = ?
        ORDER BY r.created_at DESC
    ");
    $stmt->execute([$movieId]);
    return $stmt->fetchAll();
}

// Favorite Functions
function addToFavorites($movie_id, $user_id) {
    global $pdo;
    // Use INSERT IGNORE to prevent errors if the favorite already exists
    $stmt = $pdo->prepare("INSERT IGNORE INTO favorites (movie_id, user_id) VALUES (?, ?)");
    return $stmt->execute([$movie_id, $user_id]);
}

function removeFromFavorites($movie_id, $user_id) {
    global $pdo;
    $stmt = $pdo->prepare("DELETE FROM favorites WHERE movie_id = ? AND user_id = ?");
    return $stmt->execute([$movie_id, $user_id]);
}

function getUserFavorites($userId) {
    global $pdo;
    // Fetch user favorites along with genres and average rating
    $stmt = $pdo->prepare("
        SELECT
            m.*,
            GROUP_CONCAT(mg.genre) as genres,
            (SELECT AVG(rating) FROM reviews WHERE movie_id = m.movie_id) as average_rating
        FROM favorites f
        JOIN movies m ON f.movie_id = m.movie_id
        LEFT JOIN movie_genres mg ON m.movie_id = mg.movie_id
        WHERE f.user_id = ?
        GROUP BY m.movie_id
        ORDER BY f.created_at DESC
    ");
    $stmt->execute([$userId]);
    return $stmt->fetchAll();
}

// Helper function (defined in config.php, but included here for clarity if using database.php standalone)
// function getMovieAverageRating($movieId) { /* ... implementation ... */ }
// function isMovieFavorited($movieId, $userId) { /* ... implementation ... */ }


?>
```

**3. `autentikasi/form-login.php` (Modified)**

Integrate with `includes/config.php`, use session messages for feedback, and ensure CAPTCHA is handled server-side.

```php
<?php
// autentikasi/form-login.php
require_once '../includes/config.php'; // Include config.php first

// Redirect if already authenticated
if (isAuthenticated()) {
    header('Location: ../beranda/index.php');
    exit;
}

// --- Logika generate CAPTCHA (Server-side) ---
// Generate CAPTCHA new if not set or needed after POST error
if (!isset($_SESSION['captcha_code']) || ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_SESSION['error']))) {
     $_SESSION['captcha_code'] = generateRandomString(6);
}


// --- Proses Form Login ---
$error_message = null;
$success_message = null;

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // Ambil dan bersihkan input
    $username_input = trim($_POST['username'] ?? '');
    $password_input = $_POST['password'] ?? '';
    $captcha_input = trim($_POST['captcha_input'] ?? '');

    // --- Validasi Server-side (Basic) ---
    if (empty($username_input) || empty($password_input) || empty($captcha_input)) {
        $_SESSION['error'] = 'Username/Email, Password, and CAPTCHA are required.';
        header('Location: form-login.php'); // Redirect back to show error and new CAPTCHA
        exit;
    }

    // --- Validasi CAPTCHA (Server-side) ---
    if (!isset($_SESSION['captcha_code']) || strtolower($captcha_input) !== strtolower($_SESSION['captcha_code'])) {
        $_SESSION['error'] = 'Invalid CAPTCHA.';
        header('Location: form-login.php'); // Redirect back to show error and new CAPTCHA
        exit; // Stop execution if CAPTCHA is wrong
    }

    // CAPTCHA valid, unset it immediately to prevent reuse
    unset($_SESSION['captcha_code']);


    // --- Lanjutkan proses login ---
    try {
        // Cek pengguna berdasarkan username atau email
        $stmt = $pdo->prepare("SELECT user_id, password FROM users WHERE username = ? OR email = ? LIMIT 1");
        $stmt->execute([$username_input, $username_input]);
        $user = $stmt->fetch(PDO::FETCH_ASSOC);

        // Verifikasi password
        if ($user && password_verify($password_input, $user['password'])) {
            // Password correct, set session user_id
            $_SESSION['user_id'] = $user['user_id'];

            // Regenerate session ID after successful login to prevent Session Fixation Attacks
            session_regenerate_id(true);

            // Redirect to intended URL if set, otherwise to beranda
            $redirect_url = '../beranda/index.php';
            if (isset($_SESSION['intended_url'])) {
                $redirect_url = $_SESSION['intended_url'];
                unset($_SESSION['intended_url']); // Clear the intended URL
            }

            header('Location: ' . $redirect_url);
            exit;

        } else {
            // Username/Email or password incorrect
            $_SESSION['error'] = 'Incorrect Username/Email or password.';
             // CAPTCHA already regenerated at the top if there was an error before CAPTCHA check
             // or specifically regenerated here if CAPTCHA was correct but credentials wrong
            if (!isset($_SESSION['captcha_code'])) {
                 $_SESSION['captcha_code'] = generateRandomString(6);
            }
            header('Location: form-login.php');
            exit;
        }

    } catch (PDOException $e) {
        error_log("Database error during login: " . $e->getMessage());
        $_SESSION['error'] = 'An internal error occurred. Please try again.';
         if (!isset($_SESSION['captcha_code'])) {
             $_SESSION['captcha_code'] = generateRandomString(6);
         }
        header('Location: form-login.php');
        exit;
    }
}

// Get messages from session
if (isset($_SESSION['error'])) {
    $error_message = $_SESSION['error'];
    unset($_SESSION['error']);
}
if (isset($_SESSION['success'])) {
    $success_message = $_SESSION['success'];
    unset($_SESSION['success']);
}

// Get CAPTCHA code from session for client-side drawing
$captchaCodeForClient = $_SESSION['captcha_code'] ?? '';
?>
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rate Tales - Login</title>
    <link rel="icon" type="image/png" href="../gambar/Untitled142_20250310223718.png" sizes="16x16"> <!-- Corrected path -->
    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;600&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="style.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <!-- Google Sign-In - Keep for now, although not implemented -->
    <script src="https://accounts.google.com/gsi/client" async defer></script>
</head>
<body>
    <div class="form-container login-form">
        <h2>Login</h2>

        <?php if ($error_message): ?>
            <p class="error-message"><?php echo htmlspecialchars($error_message); ?></p>
        <?php endif; ?>

         <?php if ($success_message): ?>
            <p class="success-message"><?php echo htmlspecialchars($success_message); ?></p>
        <?php endif; ?>

        <form action="form-login.php" method="POST">
            <div class="input-group">
                <label for="username">Username or Email</label>
                <input type="text" name="username" id="username" placeholder="Username or Email" required value="<?php echo htmlspecialchars($username_input ?? ''); ?>">
            </div>
            <div class="input-group">
                <label for="password">Password</label>
                <input type="password" name="password" id="password" placeholder="Password" required>
            </div>

            <div class="remember-me">
                <input type="checkbox" name="remember_me" id="rememberMe">
                <label for="rememberMe">Remember Me</label>
            </div>

             <div class="input-group">
                 <label>Verify CAPTCHA</label>
                 <div class="captcha-container">
                    <canvas id="captchaCanvas" width="150" height="40"></canvas>
                    <button type="button" onclick="generateCaptcha()" class="btn-reload" title="Reload CAPTCHA"><i class="fas fa-sync-alt"></i></button>
                 </div>
                 <input type="text" name="captcha_input" id="captchaInput" placeholder="Enter CAPTCHA" required autocomplete="off"> <!-- Remove value prefill for security -->
                 <p id="captchaMessage" class="error-message" style="display:none;"></p>
            </div>

            <button type="submit" class="btn">Login</button>
        </form>
        <br>
        <!-- Google Sign-In elements -->
        <!-- Replace YOUR_GOOGLE_CLIENT_ID -->
        <div id="g_id_onload" data-client_id="YOUR_GOOGLE_CLIENT_ID" data-callback="handleCredentialResponse"></div>
        <div class="g_id_signin" data-type="standard"></div>


        <p class="form-link">Don't have an account? <a href="form-register.php">Register here</a></p>
    </div>

    <script src="animation.js"></script>
    <script>
        // Variabel untuk menyimpan kode CAPTCHA saat ini
        // Menggunakan PHP untuk memasukkan kode dari sesi
        let currentCaptchaCode = "<?php echo htmlspecialchars($captchaCodeForClient); ?>";

        const captchaInput = document.getElementById('captchaInput');
        const captchaMessage = document.getElementById('captchaMessage');
        const captchaCanvas = document.getElementById('captchaCanvas');


        function drawCaptcha(code) {
            if (!captchaCanvas) return;
            const ctx = captchaCanvas.getContext('2d');
            ctx.clearRect(0, 0, captchaCanvas.width, captchaCanvas.height);
            ctx.fillStyle = "#0a192f"; // Background color
            ctx.fillRect(0, 0, captchaCanvas.width, captchaCanvas.height);

            ctx.font = "24px Arial";
            ctx.fillStyle = "#00e4f9"; // Text color
            ctx.strokeStyle = "#00bcd4"; // Noise line color
            ctx.lineWidth = 1;

            // Draw random lines
            for (let i = 0; i < 5; i++) {
                 ctx.beginPath();
                 ctx.moveTo(Math.random() * captchaCanvas.width, Math.random() * captchaCanvas.height);
                 ctx.lineTo(Math.random() * captchaCanvas.width, Math.random() * captchaCanvas.height);
                 ctx.stroke();
            }

            // Draw CAPTCHA text with slight variations
            const textStartX = 10;
            const textY = 30;
            const charSpacing = 23;

            ctx.save();
            ctx.translate(textStartX, textY);

            for (let i = 0; i < code.length; i++) {
                ctx.save();
                const angle = (Math.random() * 20 - 10) * Math.PI / 180;
                ctx.rotate(angle);
                const offsetX = Math.random() * 5 - 2.5;
                const offsetY = Math.random() * 5 - 2.5;
                ctx.fillText(code[i], offsetX + i * charSpacing, offsetY);
                ctx.restore();
            }
            ctx.restore();
        }

        // Function to generate new CAPTCHA (using Fetch API)
        async function generateCaptcha() {
            try {
                const response = await fetch('generate_captcha.php');
                if (!response.ok) {
                     throw new Error('Failed to load new CAPTCHA (status: ' + response.status + ')');
                 }
                const newCaptchaCode = await response.text();
                currentCaptchaCode = newCaptchaCode; // Update global variable
                drawCaptcha(currentCaptchaCode); // Redraw canvas
                captchaInput.value = ''; // Clear input field
                captchaMessage.style.display = 'none'; // Hide feedback message
            } catch (error) {
                console.error("Error generating CAPTCHA:", error);
                captchaMessage.innerText = 'Failed to load CAPTCHA. Try again.';
                captchaMessage.style.color = 'red';
                captchaMessage.style.display = 'block';
            }
        }

        // Initial drawing
        document.addEventListener('DOMContentLoaded', () => {
            drawCaptcha(currentCaptchaCode);
        });

        // Google Sign-In handler (placeholder)
        function handleCredentialResponse(response) {
           console.log("Encoded JWT ID token: " + response.credential);
           // TODO: Send this token to your server for validation
           // Example: window.location.href = 'verify-google-token.php?token=' + response.credential;
        }
    </script>
</body>
</html>
```

**4. `autentikasi/form-register.php` (Modified)**

Integrate with `includes/config.php`, use session messages for feedback, and handle DB insert.

```php
<?php
// autentikasi/form-register.php
require_once '../includes/config.php'; // Include config.php first

// Redirect if already authenticated
if (isAuthenticated()) {
    header('Location: ../beranda/index.php');
    exit;
}

// --- Logika generate CAPTCHA (Server-side) ---
// Generate CAPTCHA new if not set or needed after POST error
if (!isset($_SESSION['captcha_code']) || ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_SESSION['error']))) {
     $_SESSION['captcha_code'] = generateRandomString(6);
}


// --- Proses Form Register ---
$error_message = null;
$success_message = null;

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // Ambil dan bersihkan input
    $full_name = trim($_POST['full_name'] ?? '');
    $username = trim($_POST['username'] ?? '');
    $age_input = $_POST['age'] ?? '';
    $gender = $_POST['gender'] ?? '';
    $email = trim($_POST['email'] ?? '');
    $password = $_POST['password'] ?? '';
    $confirm_password = $_POST['confirm_password'] ?? '';
    $captcha_input = trim($_POST['captcha_input'] ?? ''); // Trim captcha input too
    $agree = isset($_POST['agree']);

    // --- Validasi Server-side ---
    $errors = [];

    if (empty($full_name)) $errors[] = 'Full Name is required.';
    if (empty($username)) $errors[] = 'Username is required.';
    if (empty($email)) $errors[] = 'Email is required.';
    if (empty($password)) $errors[] = 'Password is required.';
    if (empty($confirm_password)) $errors[] = 'Confirm Password is required.';
    if (empty($captcha_input)) $errors[] = 'CAPTCHA is required.';
    if (!$agree) $errors[] = 'You must agree to the User Agreement.';

    // Validate Age: check if numeric and > 0
    $age = filter_var($age_input, FILTER_VALIDATE_INT);
    if ($age === false || $age <= 0) {
        $errors[] = 'Invalid Age.';
    }

    // Validate Gender: check against allowed options
    $allowed_genders = ['Laki-laki', 'Perempuan'];
    if (!in_array($gender, $allowed_genders)) {
        $errors[] = 'Invalid Gender selection.';
    }

    if ($password !== $confirm_password) {
        $errors[] = 'Password and Confirm Password do not match.';
    }

    if (strlen($password) < 6) {
        $errors[] = 'Password must be at least 6 characters long.';
    }

    // Validate Email Format (basic)
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
         $errors[] = 'Invalid Email format.';
    }

    // Validate Username format (optional: allow only letters, numbers, underscore)
    // if (!preg_match('/^[a-zA-Z0-9_]+$/', $username)) {
    //     $errors[] = 'Username can only contain letters, numbers, and underscores.';
    // }


    // --- Validasi CAPTCHA (Server-side) - This is the crucial security check ---
    if (!isset($_SESSION['captcha_code']) || strtolower($captcha_input) !== strtolower($_SESSION['captcha_code'])) {
        $errors[] = 'Invalid CAPTCHA.';
        // Regenerate CAPTCHA immediately on CAPTCHA failure
        $_SESSION['captcha_code'] = generateRandomString(6);
        // Do NOT unset CAPTCHA here if it's invalid
    } else {
        // CAPTCHA valid, unset it immediately to prevent reuse
        unset($_SESSION['captcha_code']);
    }


    // If there are validation errors, store them in session and redirect
    if (!empty($errors)) {
        $_SESSION['error'] = implode('<br>', $errors);
        // Ensure CAPTCHA is regenerated if it wasn't already due to CAPTCHA failure
        if (!isset($_SESSION['captcha_code'])) {
             $_SESSION['captcha_code'] = generateRandomString(6);
        }
        header('Location: form-register.php');
        exit;
    }

    // --- If all validations pass, proceed to DB checks and INSERT ---

    // Check if username or email already exists
    try {
        $stmt_check = $pdo->prepare("SELECT COUNT(*) FROM users WHERE username = ? OR email = ? LIMIT 1"); // Use LIMIT 1
        $stmt_check->execute([$username, $email]);
        if ($stmt_check->fetchColumn() > 0) {
            $_SESSION['error'] = 'Username or email is already registered.';
             // Regenerate CAPTCHA on database check failure
            $_SESSION['captcha_code'] = generateRandomString(6);
            header('Location: form-register.php');
            exit;
        }

        // Hash password
        $hashed_password = password_hash($password, PASSWORD_BCRYPT);

        // Save new user to database
        // Use the updated createUser function
        $inserted = createUser($full_name, $username, $email, $hashed_password, $age, $gender);

        if ($inserted) {
             // Get the ID of the newly created user
            $userId = $pdo->lastInsertId();
            $_SESSION['user_id'] = $userId;

            // Regenerate session ID after successful registration
            session_regenerate_id(true);

            $_SESSION['success'] = 'Registration successful! Welcome, ' . htmlspecialchars($username) . '!';

            // Redirect to intended URL if set, otherwise to beranda
            $redirect_url = '../beranda/index.php';
            if (isset($_SESSION['intended_url'])) {
                $redirect_url = $_SESSION['intended_url'];
                unset($_SESSION['intended_url']); // Clear the intended URL
            }

            header('Location: ' . $redirect_url); // Redirect to beranda or intended page
            exit;
        } else {
             // This else block might be hit if execute returns false for other reasons
            $_SESSION['error'] = 'Failed to create user account.';
            if (!isset($_SESSION['captcha_code'])) {
                $_SESSION['captcha_code'] = generateRandomString(6);
            }
            header('Location: form-register.php');
            exit;
        }


    } catch (PDOException $e) {
        error_log("Database error during registration: " . $e->getMessage());
        $_SESSION['error'] = 'An internal error occurred during registration. Please try again.';
         if (!isset($_SESSION['captcha_code'])) {
             $_SESSION['captcha_code'] = generateRandomString(6);
         }
        header('Location: form-register.php');
        exit;
    }
}

// Get messages from session
if (isset($_SESSION['error'])) {
    $error_message = $_SESSION['error'];
    unset($_SESSION['error']);
}
if (isset($_SESSION['success'])) {
    $success_message = $_SESSION['success'];
    unset($_SESSION['success']);
}

// Get CAPTCHA code from session for client-side drawing
$captchaCodeForClient = $_SESSION['captcha_code'] ?? '';
?>
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="icon" type="image/png" href="../gambar/Untitled142_20250310223718.png"> <!-- Corrected path -->
    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;600&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="style.css">
     <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <title>Rate Tales - Register</title>
</head>
<body>
    <div class="form-container register-form">
        <h2>Register Account</h2>

        <?php if ($error_message): ?>
            <p class="error-message"><?php echo htmlspecialchars($error_message); ?></p>
        <?php endif; ?>

         <?php if ($success_message): ?>
            <p class="success-message"><?php echo htmlspecialchars($success_message); ?></p>
        <?php endif; ?>


        <form method="POST" action="form-register.php" onsubmit="return validateForm()">
            <div class="input-group">
                <label for="name">Full Name</label>
                <input type="text" id="name" name="full_name" placeholder="Enter your full name" required value="<?php echo htmlspecialchars($_POST['full_name'] ?? ''); ?>">
            </div>
            <div class="input-group">
                <label for="username">Username</label>
                <input type="text" id="username" name="username" placeholder="Enter a username" required value="<?php echo htmlspecialchars($_POST['username'] ?? ''); ?>">
            </div>
            <div class="input-group">
                <label for="usia">Age</label>
                <input type="number" id="usia" name="age" placeholder="Your age" required min="1" value="<?php echo htmlspecialchars($_POST['age'] ?? ''); ?>">
            </div>
            <div class="input-group">
                <label for="gender">Gender</label>
                <select id="gender" name="gender" required>
                    <option value="">Select...</option>
                    <option value="Laki-laki" <?php echo (($_POST['gender'] ?? '') === 'Laki-laki') ? 'selected' : ''; ?>>Male</option>
                    <option value="Perempuan" <?php echo (($_POST['gender'] ?? '') === 'Perempuan') ? 'selected' : ''; ?>>Female</option>
                </select>
            </div>
            <div class="input-group">
                <label for="email">Email</label>
                <input type="email" id="email" name="email" placeholder="Enter your email" required value="<?php echo htmlspecialchars($_POST['email'] ?? ''); ?>">
            </div>
            <div class="input-group">
                <label for="password">Create Password</label>
                <input type="password" id="password" name="password" placeholder="Create your password" required minlength="6">
            </div>
            <div class="input-group">
                <label for="confirm-password">Confirm Password</label>
                <input type="password" id="confirm-password" name="confirm_password" placeholder="Repeat your password" required>
            </div>

            <div class="input-group">
                 <label>Verify CAPTCHA</label>
                 <div class="captcha-container">
                    <canvas id="captchaCanvas" width="150" height="40"></canvas>
                    <button type="button" onclick="generateCaptcha()" class="btn-reload" title="Reload CAPTCHA"><i class="fas fa-sync-alt"></i></button>
                 </div>
                 <input type="text" name="captcha_input" id="captchaInput" placeholder="Enter CAPTCHA" required autocomplete="off"> <!-- Remove value prefill for security -->
                 <p id="captchaMessage" class="error-message" style="display:none;"></p>
            </div>

            <div style="text-align: center; margin-bottom: 10px; margin-top: 20px;">
                <button type="button" id="agreement-btn">Read User Agreement</button>
            </div>

            <div class="input-group agreement-checkbox">
                <input type="checkbox" id="agree-checkbox" name="agree" required <?php echo (isset($_POST['agree']) && $_POST['agree']) ? 'checked' : ''; ?>>
                <label for="agree-checkbox">I agree to the <a href="#" id="show-agreement-link">User Agreement</a></label>
            </div>

            <button type="submit" class="btn" id="register-submit-btn" disabled>Register</button> <!-- Disabled by default -->
        </form>
        <p class="form-link">Already have an account? <a href="form-login.php">Login here</a></p>
    </div>

    <!-- Agreement Modal HTML -->
    <div id="agreement-modal">
        <div>
            <h3>User Agreement</h3>
            <p>
                <h5><b>Privacy Policy</b></h5>
                This Privacy Policy explains how Rate Tales (“we”) collects, stores, uses, and protects your personal data during your use of this site. All data management activities are carried out in accordance with the provisions of the Law of the Republic of Indonesia Number 27 of 2022 concerning Personal Data Protection (UU PDP). By using this site and registering your account, you provide explicit consent to us to process your personal data as described in this policy.
                We may collect personal information directly when you register or use features on the site, such as full name, email address, and information related to your activity on the site, including viewing preferences, reviews, ratings, and interaction history. All data we collect is used for legitimate and proportionate purposes, namely to improve your experience using our services. We use it to provide personalized features, provide content recommendations, perform internal analysis, and - with your consent - deliver promotional information or relevant content.
                Your personal data will be stored as long as your account is active, or as long as necessary to support the service objectives. We implement appropriate technical and organizational measures to protect your data from unauthorized access, leakage, or misuse. We will not share your personal data with third parties without explicit consent from you, unless required by law or in the context of law enforcement and other legal obligations.
                In accordance with the provisions of the UU PDP, you as the data owner have the right to access your personal data, request correction or deletion of data, withdraw consent for data processing, and object to certain processing. We respect these rights and will follow up on every request you submit through the official contact channels available on our site.
                We may update the content of this Privacy Policy from time to time, especially if there are changes in regulations or technological developments that affect how we process your personal data. Significant changes will be communicated through notifications on the site or email. By continuing to use our services after the changes are enforced, you are considered to have agreed to the updated policy.
                If you have questions, requests, or complaints related to this policy or the use of your personal data, you can contact us through the official email address or contact form available on the site. By using this site, you declare that you have read, understood, and agreed to the contents of this Privacy Policy and give explicit consent for the collection and processing of your personal data by us.
            </p>
            <button id="close-agreement" class="btn">Close</button>
        </div>
    </div>


    <script src="animation.js"></script>
    <script>
        // Variable to store the current CAPTCHA code on the client side
        let currentCaptchaCode = "<?php echo htmlspecialchars($captchaCodeForClient); ?>";

        const captchaInput = document.getElementById('captchaInput');
        const captchaMessage = document.getElementById('captchaMessage');
        const captchaCanvas = document.getElementById('captchaCanvas');


        function drawCaptcha(code) {
            if (!captchaCanvas) return;
            const ctx = captchaCanvas.getContext('2d');
            ctx.clearRect(0, 0, captchaCanvas.width, captchaCanvas.height);
            ctx.fillStyle = "#0a192f";
            ctx.fillRect(0, 0, captchaCanvas.width, captchaCanvas.height);

            ctx.font = "24px Arial";
            ctx.fillStyle = "#00e4f9";
            ctx.strokeStyle = "#00bcd4";
            ctx.lineWidth = 1;

            for (let i = 0; i < 5; i++) {
                ctx.beginPath();
                ctx.moveTo(Math.random() * captchaCanvas.width, Math.random() * captchaCanvas.height);
                ctx.lineTo(Math.random() * captchaCanvas.width, Math.random() * captchaCanvas.height);
                ctx.stroke();
            }

            const textStartX = 10;
            const textY = 30;
            const charSpacing = 23;

            ctx.save();
            ctx.translate(textStartX, textY);

            for (let i = 0; i < code.length; i++) {
                ctx.save();
                const angle = (Math.random() * 20 - 10) * Math.PI / 180;
                ctx.rotate(angle);
                const offsetX = Math.random() * 5 - 2.5;
                const offsetY = Math.random() * 5 - 2.5;
                ctx.fillText(code[i], offsetX + i * charSpacing, offsetY);
                ctx.restore();
            }
            ctx.restore();
        }

        // Function to generate new CAPTCHA (using Fetch API)
        async function generateCaptcha() {
            try {
                const response = await fetch('generate_captcha.php');
                 if (!response.ok) {
                     throw new Error('Failed to load new CAPTCHA (status: ' + response.status + ')');
                 }
                const newCaptchaCode = await response.text();
                currentCaptchaCode = newCaptchaCode;
                drawCaptcha(currentCaptchaCode);
                captchaInput.value = '';
                captchaMessage.style.display = 'none';
            } catch (error) {
                console.error("Error generating CAPTCHA:", error);
                captchaMessage.innerText = 'Failed to load CAPTCHA. Try again.';
                captchaMessage.style.color = 'red';
                captchaMessage.style.display = 'block';
            }
        }

        // Initial drawing
        document.addEventListener('DOMContentLoaded', () => {
            drawCaptcha(currentCaptchaCode);

             // Set initial state of the Register button based on the agreement checkbox
             const agreeCheckbox = document.getElementById('agree-checkbox');
             const registerSubmitBtn = document.getElementById('register-submit-btn');
             if(registerSubmitBtn && agreeCheckbox) {
                 registerSubmitBtn.disabled = !agreeCheckbox.checked;
             }
        });


        // Client-side form validation
        function validateForm() {
            const password = document.getElementById('password').value;
            const confirmPassword = document.getElementById('confirm-password').value;
            const agreeCheckbox = document.getElementById('agree-checkbox');

            // Clear previous client-side messages
            const feedbackElement = document.getElementById('captchaMessage'); // Use this element for feedback
            if (feedbackElement) {
                 feedbackElement.style.display = 'none';
                 feedbackElement.innerText = '';
                 feedbackElement.style.color = 'red'; // Reset color
            }


            if (password !== confirmPassword) {
                 if (feedbackElement) {
                     feedbackElement.innerText = 'Password and Confirm Password do not match!';
                     feedbackElement.style.display = 'block';
                 } else {
                    alert('Password and Confirm Password do not match!');
                 }
                return false; // Prevent form submission
            }

             if (!agreeCheckbox.checked) {
                 if (feedbackElement) {
                     feedbackElement.innerText = 'You must agree to the User Agreement.';
                     feedbackElement.style.display = 'block';
                 } else {
                     alert('You must agree to the User Agreement.');
                 }
                 return false; // Prevent form submission
            }

            // Client-side CAPTCHA check (optional, server-side is mandatory)
            // if (captchaInput.value.toLowerCase() !== currentCaptchaCode.toLowerCase()) {
            //      if (feedbackElement) {
            //          feedbackElement.innerText = 'Invalid CAPTCHA!';
            //          feedbackElement.style.display = 'block';
            //      } else {
            //          alert('Invalid CAPTCHA!');
            //      }
            //     generateCaptcha(); // Regenerate CAPTCHA on client-side failure
            //     return false; // Prevent form submission
            // }

            // If all client-side checks pass, allow form submission
            // Server-side validation (including CAPTCHA) will run upon POST
            return true;
        }

        // Modal Agreement Logic
        const agreementBtn = document.getElementById('agreement-btn');
        const showAgreementLink = document.getElementById('show-agreement-link');
        const agreementModal = document.getElementById('agreement-modal');
        const closeAgreement = document.getElementById('close-agreement');
        const agreeCheckbox = document.getElementById('agree-checkbox');
        const registerSubmitBtn = document.getElementById('register-submit-btn');


        function showAgreementModal() {
            if(agreementModal) {
                 agreementModal.style.display = 'flex'; // Use flex to center
            }
        }

        if(agreementBtn) agreementBtn.addEventListener('click', showAgreementModal);
        if(showAgreementLink) showAgreementLink.addEventListener('click', (e) => {
            e.preventDefault();
            showAgreementModal();
        });
        if(closeAgreement) closeAgreement.addEventListener('click', () => {
            if(agreementModal) agreementModal.style.display = 'none';
        });

        // Close modal if clicking outside content
        if(agreementModal) {
            agreementModal.addEventListener('click', (e) => {
                if (e.target === agreementModal) {
                    agreementModal.style.display = 'none';
                }
            });
        }

        // Enable/disable Register button based on agreement checkbox
        if(agreeCheckbox && registerSubmitBtn) {
            agreeCheckbox.addEventListener('change', () => {
                registerSubmitBtn.disabled = !agreeCheckbox.checked;
            });
        }
    </script>
</body>
</html>
```

**5. `autentikasi/generate_captcha.php` (No Changes Needed)**

This file is correct as is, assuming `../includes/config.php` provides `session_start()` and `generateRandomString()`.

**6. `autentikasi/logout.php` (No Changes Needed)**

This file is correct as is.

**7. `autentikasi/styles.css` (Modified)**

Minor adjustments for consistency and potentially refining CAPTCHA area.

```css
/* style.css */

* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: 'Poppins', sans-serif;
}

body {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
    background: url(../gambar/5302920.jpg) no-repeat center center fixed; /* Corrected path */
    background-size: cover;
}

/* General form container styling */
.form-container {
    background: linear-gradient(to right, #0f2027, #203a43, #2c5364);
    padding: 30px;
    width: 100%;
    max-width: 400px;
    border-radius: 12px;
    box-shadow: 0 10px 30px rgba(0, 255, 255, 0.2);
    opacity: 0; /* Initial state for JS animation */
    transform: translateY(-50px); /* Initial state for JS animation */
    transition: opacity 1s ease-in-out, transform 1s ease-in-out; /* Animation */
    position: relative; /* Needed for Google button positioning if it becomes absolute */
}

/* Class added by JS to show the form */
.form-container.show {
    opacity: 1;
    transform: translateY(0);
}


h2 {
    text-align: center;
    margin-bottom: 20px;
    color: #00e4f9; /* Teal color */
}

/* General input group styling */
.input-group {
    margin-bottom: 15px;
}

/* Override for checkbox width */
.input-group input[type="checkbox"] {
    width: auto;
    margin-right: 5px;
    padding: 0;
    margin-top: 0;
    margin-bottom: 0;
    vertical-align: middle;
}

label {
    font-weight: 600;
    display: block;
    margin-bottom: 5px;
    color: #b0e0e6; /* Light grayish blue */
}

input:not([type="checkbox"]),
select {
    width: 100%;
    padding: 10px;
    border: 1px solid #00bcd4; /* Teal border */
    border-radius: 5px;
    background: #0a192f; /* Dark background */
    color: #e0e0e0; /* Light text color */
    transition: border-color 0.3s, box-shadow 0.3s;
    font-size: 16px;
}

/* Placeholder color */
input::placeholder {
    color: #737878; /* Grayish placeholder */
    opacity: 1;
}

input:focus,
select:focus {
    border-color: #00e4f9; /* Brighter teal focus */
    box-shadow: 0 0 5px rgba(0, 228, 249, 0.8); /* Teal glow */
    outline: none;
}

/* General button styling */
.btn {
    width: 100%;
    background: #00bcd4; /* Teal button */
    color: white;
    padding: 10px;
    border: none;
    border-radius: 5px;
    font-size: 16px;
    cursor: pointer;
    transition: background 0.3s, box-shadow 0.3s;
    margin-top: 10px;
}

.btn:hover {
    background: #00e4f9; /* Brighter teal on hover */
    box-shadow: 0 0 10px rgba(0, 228, 249, 0.5); /* Teal glow on hover */
}

.btn:disabled {
    background: #555;
    cursor: not-allowed;
    box-shadow: none;
}


/* General form link styling */
.form-link {
    text-align: center;
    margin-top: 15px;
    font-size: 14px;
    color: #b0e0e6;
}

.form-link a {
    color: #00e4f9; /* Teal link color */
    text-decoration: none;
    font-weight: bold;
}

.form-link a:hover {
    text-decoration: underline;
}

/* Specific styling for CAPTCHA container */
.captcha-container {
    color: #b0e0e6;
    display: flex;
    align-items: center;
    gap: 10px;
    margin-top: 5px;
    background-color: #1a1a1a; /* Dark background for container */
    border: 1px solid #363636; /* Border matching other inputs loosely */
    border-radius: 8px; /* Rounded corners */
    padding: 5px; /* Padding around canvas and button */
}

.captcha-container canvas {
     border: 1px solid #00e4f9; /* Teal border for canvas */
     border-radius: 5px; /* Match input border radius */
     background-color: #2c3e50; /* Dark blue background for canvas */
     flex-shrink: 0; /* Prevent canvas from shrinking */
}

.captcha-container .btn-reload {
    width: auto;
    padding: 5px 10px;
    margin-top: 0; /* Remove default margin-top from .btn */
    font-size: 14px;
    background: #2c5364; /* Darker blue background */
    color: #b0e0e6; /* Light text color */
    flex-shrink: 0; /* Prevent button from shrinking */
}

.captcha-container .btn-reload:hover {
    background: #3a6374;
    box-shadow: none;
}

/* Style for CAPTCHA input field below the container */
.input-group input[name="captcha_input"] {
    margin-top: 5px; /* Add space above input */
}


/* Specific style for Remember Me (Login) */
.remember-me {
    display: flex;
    align-items: center;
    justify-content: flex-start;
    margin: 10px 0;
    color: #b0e0e6;
}

/* Specific style for Agreement Modal (Register) */
#agreement-modal {
    display: none;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0,0,0,0.8); /* Darker overlay */
    justify-content: center;
    align-items: center;
    z-index: 999;
    padding: 20px;
}

#agreement-modal > div {
    background: #1a1a1a; /* Dark background */
    padding: 30px;
    border-radius: 15px; /* More rounded */
    width: 100%;
    max-width: 600px;
    color: #e0e0e0;
    max-height: 90vh;
    overflow-y: auto;
    position: relative;
    box-shadow: 0 5px 20px rgba(0, 0, 0, 0.5);
}

#agreement-modal h3 {
    color: #00ffff; /* Teal header */
    margin-bottom: 20px;
    text-align: center;
}

#agreement-modal h5 {
    color: #b0e0e6; /* Light grayish blue sub-header */
    margin-top: 15px;
    margin-bottom: 5px;
}

#agreement-modal p {
    font-size: 14px;
    line-height: 1.6;
    color: #ccc; /* Slightly lighter text */
}

#agreement-modal .btn {
    /* Close button style */
    width: auto;
    padding: 10px 20px;
    margin-top: 20px;
    display: block;
    margin-left: auto;
    margin-right: auto;
    background-color: #00ffff; /* Teal button */
    color: #1a1a1a; /* Dark text */
}
#agreement-modal .btn:hover {
     background-color: #00cccc; /* Darker teal on hover */
     box-shadow: none; /* Remove glow */
}


/* Style for link "User Agreement" in the label */
label a#show-agreement-link {
    color: #00e4f9;
    text-decoration: underline;
    font-weight: bold;
}
label a#show-agreement-link:hover {
    text-decoration: none;
}

/* Style for button "Read User Agreement" */
button#agreement-btn {
     background: none;
     border: none;
     font-size: 14px; /* Smaller font size */
     color: #00e4f9;
     cursor: pointer;
     padding: 5px 10px; /* Smaller padding */
     margin: 0;
     text-align: center;
     display: inline-block;
     vertical-align: middle;
     text-decoration: underline; /* Explicitly underline */
     transition: color 0.3s;
}
button#agreement-btn:hover {
    color: #00cccc;
    text-decoration: none;
}


/* Style for the div containing the agreement checkbox and label */
.input-group.agreement-checkbox {
    display: flex;
    align-items: flex-start;
    gap: 10px;
    margin-bottom: 20px; /* Add more space below checkbox */
}
.input-group.agreement-checkbox label {
     display: inline-block;
     margin-bottom: 0;
     font-weight: normal;
     flex-grow: 1;
     line-height: 1.4;
     color: #b0e0e6; /* Consistent label color */
}
```

**8. `beranda/index.php` (Modified)**

Fetch movies from the database and display user info if logged in. Update navigation links.

```php
<?php
// beranda/index.php
require_once '../includes/config.php'; // Include config.php

// Fetch all movies from the database
$movies = getAllMovies();

// Get authenticated user details (if logged in)
$user = getAuthenticatedUser();

// Determine movies for sections (simple logic for now)
$featured_movies = array_slice($movies, 0, 5); // First 5 for slider
$trending_movies = array_slice($movies, 0, 10); // First 10 for trending
$for_you_movies = array_slice($movies, 1, 10); // A different slice for variety
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Home</title>
    <link rel="stylesheet" href="styles.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Using FA 6 for consistency -->
</head>
<body>
    <div class="container">
        <nav class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li class="active"><a href="#"><i class="fas fa-home"></i> <span>Home</span></a></li>
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favourites</span></a></li>
                <li><a href="../review/index.php"><i class="fas fa-star"></i> <span>Review</span></a></li>
                <li><a href="../manage/indeks.php"><i class="fas fa-film"></i> <span>Manage</span></a></li>
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Corrected logout link -->
                </ul>
            </div>
        </nav>
        <main class="main-content">
            <div class="hero-section">
                <div class="featured-movie-slider">
                    <?php if (!empty($featured_movies)): ?>
                        <?php foreach ($featured_movies as $index => $movie): ?>
                            <div class="slide <?php echo $index === 0 ? 'active' : ''; ?>">
                                <img src="<?php echo htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>"> <!-- Use poster_image -->
                                <div class="movie-info">
                                    <h1><?php echo htmlspecialchars($movie['title']); ?></h1>
                                    <p><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?></p> <!-- Use release_date year and genres -->
                                </div>
                            </div>
                        <?php endforeach; ?>
                    <?php else: ?>
                         <div class="slide active empty-state" style="position:relative; opacity:1; display:flex; flex-direction:column; justify-content:center; align-items:center;">
                             <i class="fas fa-film" style="font-size: 3em; color:#00ffff; margin-bottom:15px;"></i>
                             <p style="color: #fff; font-size:1.2em;">No featured movies available yet.</p>
                         </div>
                    <?php endif; ?>
                </div>
            </div>
            <section class="trending-section">
                <h2>TRENDING</h2>
                <div class="scroll-container">
                    <div class="movie-grid">
                        <?php if (!empty($trending_movies)): ?>
                            <?php foreach ($trending_movies as $movie): ?>
                                <div class="movie-card" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'"> <!-- Link to movie details -->
                                    <img src="<?php echo htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>"> <!-- Use poster_image -->
                                    <div class="movie-details">
                                        <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                                        <p><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?></p> <!-- Use release_date year and genres -->
                                    </div>
                                </div>
                            <?php endforeach; ?>
                        <?php else: ?>
                            <div class="empty-state" style="width: 100%; text-align: center; padding: 20px;">
                                <p style="color: #888;">No trending movies available.</p>
                            </div>
                        <?php endif; ?>
                    </div>
                </div>
            </section>
            <section class="for-you-section">
                <h2>For You</h2>
                <div class="scroll-container">
                    <div class="movie-grid">
                        <?php if (!empty($for_you_movies)): ?>
                            <?php foreach ($for_you_movies as $movie): ?>
                                <div class="movie-card" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'"> <!-- Link to movie details -->
                                    <img src="<?php echo htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>"> <!-- Use poster_image -->
                                    <div class="movie-details">
                                        <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                                        <p><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?></p> <!-- Use release_date year and genres -->
                                    </div>
                                </div>
                            <?php endforeach; ?>
                         <?php else: ?>
                            <div class="empty-state" style="width: 100%; text-align: center; padding: 20px;">
                                <p style="color: #888;">No movie suggestions for you.</p>
                            </div>
                        <?php endif; ?>
                    </div>
                </div>
            </section>
        </main>
         <?php if ($user): ?>
             <a href="../acc_page/index.php" class="user-profile"> <!-- Link to profile -->
                 <img src="<?php echo htmlspecialchars($user['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($user['full_name'] ?? $user['username']) . '&background=random'); ?>" alt="User Profile" class="profile-pic">
                 <span><?php echo htmlspecialchars($user['full_name'] ?? $user['username']); ?></span>
             </a>
         <?php else: ?>
             <!-- Optional: Link to login if not logged in -->
             <a href="../autentikasi/form-login.php" class="user-profile" style="background-color: #00ffff; color: #1a1a1a;">
                 <span>Login</span>
             </a>
         <?php endif; ?>
    </div>
    <script>
    document.addEventListener('DOMContentLoaded', function() {
        const slides = document.querySelectorAll('.hero-section .slide'); // Select slides within hero-section
        if (slides.length > 0) {
            let currentSlide = 0;

            function showSlide(index) {
                slides.forEach(slide => slide.classList.remove('active'));
                slides[index].classList.add('active');
            }

            function nextSlide() {
                currentSlide = (currentSlide + 1) % slides.length;
                showSlide(currentSlide);
            }

            // Change slide every 5 seconds
            setInterval(nextSlide, 5000);
        }
         // Smooth scroll for movie grids (optional enhancement)
         document.querySelectorAll('.scroll-container').forEach(container => {
             container.addEventListener('wheel', (e) => {
                 if (e.deltaY > 0) {
                     container.scrollLeft += 100; // Scroll right
                 } else {
                     container.scrollLeft -= 100; // Scroll left
                 }
                 e.preventDefault(); // Prevent vertical scrolling of the page
             });
         });
    });
    </script>
</body>
</html>
```

**9. `beranda/styles.css` (Modified)**

Minor adjustments for consistent teal color and layout.

```css
/* beranda/styles.css */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

body {
    background-color: #1a1a1a; /* Dark background */
    color: #ffffff; /* White text */
}

.container {
    display: flex;
    min-height: 100vh;
}

/* Sidebar Styles */
.sidebar {
    width: 250px;
    background-color: #242424; /* Dark gray sidebar */
    padding: 20px;
    display: flex;
    flex-direction: column;
    position: fixed;
    height: 100vh;
    top: 0;
    left: 0;
    z-index: 100; /* Ensure sidebar is above content */
    box-shadow: 2px 0 5px rgba(0,0,0,0.3); /* Subtle shadow */
}

.logo h2 {
    color: #00ffff; /* Teal logo color */
    margin-bottom: 40px;
    font-size: 24px;
    text-align: center; /* Center logo */
}

.nav-links {
    list-style: none;
    flex-grow: 1;
    padding-top: 20px; /* Space below logo */
}

.nav-links li, .bottom-links li {
    margin-bottom: 15px; /* Slightly reduced spacing */
}

.nav-links a, .bottom-links a {
    color: #ffffff;
    text-decoration: none;
    display: flex;
    align-items: center;
    padding: 12px 15px; /* Increased padding */
    border-radius: 8px;
    transition: background-color 0.3s, color 0.3s;
    font-size: 15px; /* Slightly smaller font */
}

.nav-links a:hover, .bottom-links a:hover {
    background-color: #363636; /* Darker gray on hover */
    color: #00ffff; /* Teal color on hover */
}

.nav-links i, .bottom-links i {
    margin-right: 15px; /* Increased icon spacing */
    width: 20px;
    text-align: center; /* Center icons */
}

.active a {
    background-color: #363636;
    color: #00ffff; /* Active color */
    font-weight: bold;
}
.active a i {
    color: #00ffff; /* Active icon color */
}


.bottom-links {
    margin-top: auto; /* Push to bottom */
    list-style: none;
    padding-top: 20px; /* Space above logout */
    border-top: 1px solid #363636; /* Separator line */
}

/* Main Content Styles */
.main-content {
    flex-grow: 1;
    margin-left: 250px; /* Match sidebar width */
    padding: 20px;
    max-width: calc(100% - 250px); /* Ensure it doesn't overflow */
    overflow-x: hidden;
    overflow-y: auto; /* Enable vertical scrolling */
    height: 100vh; /* Fill viewport height */
}

/* Hero Section */
.hero-section {
    position: relative;
    height: 450px; /* Increased height */
    margin-bottom: 40px;
    background-color: #242424; /* Placeholder background */
    border-radius: 15px;
    overflow: hidden; /* Needed for slide images */
}

.featured-movie-slider {
    position: relative;
    height: 100%;
}

.slide {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    opacity: 0;
    transition: opacity 0.8s ease-in-out; /* Slower transition */
    background-size: cover; /* Ensure background covers area */
    background-position: center;
    display: flex; /* Use flex to center content if needed */
    align-items: flex-end; /* Align info to bottom */
    justify-content: flex-start;
}

.slide.active {
    opacity: 1;
}

.slide img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    position: absolute; /* Position image behind info */
    top: 0;
    left: 0;
    z-index: 1;
}

.movie-info {
    position: relative; /* Position info above image */
    z-index: 2;
    bottom: 0;
    left: 0;
    right: 0;
    padding: 40px;
    background: linear-gradient(transparent, rgba(0, 0, 0, 0.9)); /* Gradient overlay */
}

.movie-info h1 {
    font-size: 38px; /* Larger title */
    margin-bottom: 10px;
    color: #00ffff; /* Teal title color */
    text-shadow: 2px 2px 4px rgba(0,0,0,0.5); /* Add shadow for readability */
}

.movie-info p {
    font-size: 18px;
    color: #ffffff; /* White text for info */
}

/* Movie Sections */
.trending-section, .for-you-section {
    margin-bottom: 40px;
    width: 100%;
    overflow: hidden;
    max-width: 100%;
}

.trending-section h2, .for-you-section h2 {
    margin-bottom: 20px;
    color: #00ffff; /* Teal headers */
    font-size: 24px;
}

/* Scrollable Container (using Flexbox for movie-grid) */
.scroll-container {
    width: 100%;
    overflow-x: auto; /* Enable horizontal scrolling */
    padding: 10px 0; /* Add padding for scrollbar visibility */
    scroll-behavior: smooth;
    scrollbar-width: thin;
    scrollbar-color: #00ffff #2a2a2a; /* Teal thumb, dark track */
}

.scroll-container::-webkit-scrollbar {
    height: 8px;
}

.scroll-container::-webkit-scrollbar-track {
    background: #2a2a2a; /* Dark track */
    border-radius: 4px;
}

.scroll-container::-webkit-scrollbar-thumb {
    background: #00ffff; /* Teal thumb */
    border-radius: 4px;
}

.scroll-container::-webkit-scrollbar-thumb:hover {
    background: #00cccc; /* Darker teal on hover */
}

/* Movie Grid (Flexbox for horizontal scroll) */
.movie-grid {
    display: flex; /* Use flexbox */
    gap: 20px; /* Gap between cards */
    padding: 4px 0;
    white-space: nowrap; /* Prevent wrapping */
    width: max-content; /* Allow content to define width */
}

.movie-card {
    flex: 0 0 200px; /* Do not grow or shrink, fixed width */
    min-width: 200px;
    background-color: #242424; /* Dark gray card background */
    border-radius: 10px;
    overflow: hidden;
    transition: transform 0.3s;
    cursor: pointer;
    display: inline-block; /* Ensure it respects white-space: nowrap */
    vertical-align: top; /* Align to top */
}

.movie-card:hover {
    transform: translateY(-5px); /* Lift effect on hover */
}

.movie-card img {
    width: 100%;
    height: 300px; /* Fixed height for poster */
    object-fit: cover;
}

.movie-details {
    padding: 15px;
}

.movie-details h3 {
    font-size: 16px;
    margin-bottom: 5px;
    white-space: normal; /* Allow title to wrap if needed */
    overflow: hidden;
    text-overflow: ellipsis; /* Add ellipsis if title is too long */
}

.movie-details p {
    font-size: 14px;
    color: #888; /* Gray text for meta info */
    white-space: normal;
}

/* User Profile in Header */
.user-profile {
    position: fixed;
    top: 20px;
    right: 20px;
    display: flex;
    align-items: center;
    background-color: #242424; /* Dark gray background */
    padding: 8px 15px;
    border-radius: 20px;
    cursor: pointer;
    text-decoration: none; /* Remove underline */
    color: #ffffff; /* White text */
    z-index: 200; /* Ensure it's above everything */
    transition: background-color 0.3s;
}

.user-profile:hover {
    background-color: #363636; /* Darker gray on hover */
}

.profile-pic {
    width: 30px;
    height: 30px;
    border-radius: 50%;
    margin-right: 10px;
    object-fit: cover; /* Ensure image covers area */
    border: 1px solid #00ffff; /* Subtle teal border */
}

/* Empty state for sections */
.empty-state {
    color: #888;
    text-align: center;
    padding: 20px;
}

.empty-state i {
    font-size: 2em;
    color: #666;
    margin-bottom: 10px;
}
```

**10. `favorite/index.php` (Modified)**

Fetch user favorites from the database. Implement remove favorite functionality (using a form). Update navigation.

```php
<?php
// favorite/index.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated
redirectIfNotAuthenticated();

// Get authenticated user ID
$userId = $_SESSION['user_id'];

// Fetch user's favorite movies
$movies = getUserFavorites($userId);

// Handle remove favorite action
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['remove_favorite_id'])) {
    $movieIdToRemove = filter_var($_POST['remove_favorite_id'], FILTER_VALIDATE_INT);
    if ($movieIdToRemove) {
        if (removeFromFavorites($movieIdToRemove, $userId)) {
            // Success, redirect back to favorites page to refresh the list
            $_SESSION['success_message'] = 'Movie removed from favorites.';
        } else {
            $_SESSION['error_message'] = 'Failed to remove movie from favorites.';
        }
    } else {
        $_SESSION['error_message'] = 'Invalid movie ID.';
    }
    header('Location: index.php'); // Redirect to prevent form resubmission
    exit;
}

// Get messages from session
$success_message = isset($_SESSION['success_message']) ? $_SESSION['success_message'] : null;
unset($_SESSION['success_message']);
$error_message = isset($_SESSION['error_message']) ? $_SESSION['error_message'] : null;
unset($_SESSION['error_message']);

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Favourites</title>
    <link rel="stylesheet" href="styles.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Using FA 6 -->
</head>
<body>
    <div class="container">
        <nav class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li>
                <li class="active"><a href="#"><i class="fas fa-heart"></i> <span>Favourites</span></a></li>
                <li><a href="../review/index.php"><i class="fas fa-star"></i> <span>Review</span></a></li>
                <li><a href="../manage/indeks.php"><i class="fas fa-film"></i> <span>Manage</span></a></li>
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Corrected logout link -->
                </ul>
            </div>
        </nav>
        <main class="main-content">
            <div class="review-header">
                <h1>My Favourites</h1>
                <div class="search-bar">
                    <input type="text" id="searchInput" placeholder="Search favourites...">
                    <button><i class="fas fa-search"></i></button>
                </div>
            </div>

             <?php if ($success_message): ?>
                <div class="alert success"><?php echo htmlspecialchars($success_message); ?></div>
            <?php endif; ?>
            <?php if ($error_message): ?>
                <div class="alert error"><?php echo htmlspecialchars($error_message); ?></div>
            <?php endif; ?>

            <div class="review-grid">
                <?php if (!empty($movies)): ?>
                    <?php foreach ($movies as $movie): ?>
                        <div class="movie-card">
                            <div class="movie-poster" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'"> <!-- Link to details -->
                                <img src="<?php echo htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                <div class="movie-actions">
                                    <!-- Form to remove favorite -->
                                     <form action="index.php" method="POST" onsubmit="return confirm('Are you sure you want to remove this movie from favorites?');">
                                         <input type="hidden" name="remove_favorite_id" value="<?php echo $movie['movie_id']; ?>">
                                         <button type="submit" class="action-btn" title="Remove from favourites"><i class="fas fa-trash"></i></button>
                                     </form>
                                </div>
                            </div>
                            <div class="movie-details">
                                <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                                <p class="movie-info"><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?></p>
                                <div class="rating">
                                    <div class="stars">
                                        <?php
                                        // Display average rating stars
                                        $average_rating = floatval($movie['average_rating']);
                                        for ($i = 1; $i <= 5; $i++) {
                                            if ($i <= $average_rating) {
                                                echo '<i class="fas fa-star"></i>'; // Full star
                                            } else if ($i - 0.5 <= $average_rating) {
                                                echo '<i class="fas fa-star-half-alt"></i>'; // Half star
                                            } else {
                                                echo '<i class="far fa-star"></i>'; // Empty star
                                            }
                                        }
                                        ?>
                                    </div>
                                    <span class="rating-count">(<?php echo htmlspecialchars($movie['average_rating']); ?>)</span>
                                </div>
                            </div>
                        </div>
                    <?php endforeach; ?>
                <?php else: ?>
                     <div class="empty-state full-width">
                         <i class="fas fa-heart-broken"></i>
                         <p>You haven't added any movies to favorites yet.</p>
                         <p class="subtitle">Find movies in the <a href="../review/index.php">Review</a> section and add them.</p>
                     </div>
                <?php endif; ?>
            </div>
        </main>
    </div>
    <script>
        document.addEventListener('DOMContentLoaded', function() {
            const searchInput = document.getElementById('searchInput');
            const movieCards = document.querySelectorAll('.review-grid .movie-card');
            const reviewGrid = document.querySelector('.review-grid');
            const initialEmptyState = document.querySelector('.empty-state.full-width');

            searchInput.addEventListener('input', function() {
                const searchTerm = this.value.toLowerCase();
                let visibleCardCount = 0;

                movieCards.forEach(card => {
                    const title = card.querySelector('h3').textContent.toLowerCase();
                    const info = card.querySelector('.movie-info').textContent.toLowerCase();

                    if (title.includes(searchTerm) || info.includes(searchTerm)) {
                        card.style.display = '';
                        visibleCardCount++;
                    } else {
                        card.style.display = 'none';
                    }
                });

                // Handle empty state visibility
                const searchEmptyState = document.querySelector('.search-empty-state');

                if (visibleCardCount === 0 && searchTerm !== '') {
                    if (!searchEmptyState) {
                        // Hide the initial empty state if it exists and we are searching
                        if(initialEmptyState) initialEmptyState.style.display = 'none';

                        const emptyState = document.createElement('div');
                        emptyState.className = 'empty-state search-empty-state full-width'; // Add full-width class
                        emptyState.innerHTML = `
                            <i class="fas fa-search"></i>
                            <p>No favorites found matching "${htmlspecialchars(searchTerm)}"</p>
                            <p class="subtitle">Try a different search term</p>
                        `;
                        reviewGrid.appendChild(emptyState);
                    } else {
                         // Update text if search empty state already exists
                         searchEmptyState.querySelector('p:first-of-type').innerText = `No favorites found matching "${htmlspecialchars(searchTerm)}"`;
                         searchEmptyState.style.display = 'flex'; // Ensure it's visible
                    }
                } else {
                    // Remove search empty state if cards are visible or search is cleared
                    if (searchEmptyState) {
                        searchEmptyState.remove();
                    }
                     // Show initial empty state if no movies were loaded AND search is cleared
                    if (movieCards.length === 0 && searchTerm === '' && initialEmptyState) {
                         initialEmptyState.style.display = 'flex';
                    }
                }
            });

            // Trigger search when search button is clicked
            const searchButton = document.querySelector('.search-bar button');
            searchButton.addEventListener('click', function() {
                const event = new Event('input');
                searchInput.dispatchEvent(event);
            });
        });

         // Helper function for HTML escaping (client-side)
         function htmlspecialchars(str) {
             if (typeof str !== 'string') return str;
             return str.replace(/&/g, '&amp;')
                       .replace(/</g, '&lt;')
                       .replace(/>/g, '&gt;')
                       .replace(/"/g, '&quot;')
                       .replace(/'/g, '&#039;');
         }
    </script>
</body>
</html>
```

**11. `favorite/styles.css` (Modified)**

Minor adjustments for consistency and potentially empty state styling.

```css
/* favorite/styles.css */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

body {
    background-color: #1a1a1a; /* Dark background */
    color: #ffffff; /* White text */
}

.container {
    display: flex;
    min-height: 100vh;
}

/* Sidebar Styles (Copy from beranda/styles.css for consistency) */
.sidebar {
    width: 250px;
    background-color: #242424; /* Dark gray sidebar */
    padding: 20px;
    display: flex;
    flex-direction: column;
    position: fixed;
    height: 100vh;
    top: 0;
    left: 0;
    z-index: 100;
    box-shadow: 2px 0 5px rgba(0,0,0,0.3);
}

.logo h2 {
    color: #00ffff; /* Teal logo color */
    margin-bottom: 40px;
    font-size: 24px;
     text-align: center;
}

.nav-links {
    list-style: none;
    flex-grow: 1;
     padding-top: 20px;
}

.nav-links li, .bottom-links li {
    margin-bottom: 15px;
}

.nav-links a, .bottom-links a {
    color: #ffffff;
    text-decoration: none;
    display: flex;
    align-items: center;
    padding: 12px 15px;
    border-radius: 8px;
    transition: background-color 0.3s, color 0.3s;
    font-size: 15px;
}

.nav-links a:hover, .bottom-links a:hover {
    background-color: #363636;
    color: #00ffff;
}

.nav-links i, .bottom-links i {
    margin-right: 15px;
    width: 20px;
     text-align: center;
}

.active a {
    background-color: #363636;
    color: #00ffff;
    font-weight: bold;
}
.active a i {
    color: #00ffff;
}


.bottom-links {
    margin-top: auto;
    list-style: none;
    padding-top: 20px;
    border-top: 1px solid #363636;
}


/* Main Content Styles */
.main-content {
    flex-grow: 1;
    margin-left: 250px; /* Match sidebar width */
    padding: 20px;
    max-height: 100vh;
    overflow-y: auto; /* Enable vertical scrolling */
}

/* Review Header Styles */
.review-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 30px;
    padding: 20px;
    background-color: #242424;
    border-radius: 15px;
}

.review-header h1 {
    font-size: 28px;
    color: #00ffff; /* Teal header */
}

.search-bar {
    display: flex;
    gap: 10px;
}

.search-bar input {
    padding: 10px 15px;
    border-radius: 8px;
    border: none;
    background-color: #363636; /* Darker input background */
    color: #ffffff;
    width: 300px;
    transition: border-color 0.3s, box-shadow 0.3s;
}
.search-bar input:focus {
     border-color: #00ffff;
     box-shadow: 0 0 0 2px rgba(0, 255, 255, 0.2);
     outline: none;
}

.search-bar button {
    padding: 10px 20px;
    border-radius: 8px;
    border: none;
    background-color: #00ffff; /* Teal button */
    color: #1a1a1a; /* Dark text */
    cursor: pointer;
    transition: background-color 0.3s;
}

.search-bar button:hover {
    background-color: #00cccc; /* Darker teal on hover */
}

/* Review Grid Styles */
.review-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 20px;
    padding: 20px 0; /* Remove left/right padding to align with main content */
}

.movie-card {
    background-color: #242424;
    border-radius: 15px;
    overflow: hidden;
    transition: transform 0.3s;
    display: flex; /* Use flexbox for layout */
    flex-direction: column; /* Stack items vertically */
    position: relative; /* Needed for absolute positioning of actions */
}

.movie-card:hover {
    transform: translateY(-5px);
}

.movie-poster {
    position: relative;
    width: 100%;
    padding-top: 150%; /* Aspect ratio 2:3 */
    flex-shrink: 0; /* Prevent poster from shrinking */
    cursor: pointer; /* Indicate clickable */
}

.movie-poster img {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.movie-actions {
    position: absolute;
    bottom: 0; /* Position at the bottom of the poster */
    left: 0;
    right: 0;
    padding: 15px;
    background: linear-gradient(transparent, rgba(0, 0, 0, 0.8));
    display: flex;
    justify-content: center;
    gap: 15px;
    opacity: 0;
    transition: opacity 0.3s ease-in-out;
     z-index: 10; /* Ensure actions are above poster */
}

.movie-card:hover .movie-actions {
    opacity: 1;
}

.movie-actions form { /* Style form inside movie-actions */
    margin: 0;
    padding: 0;
    display: flex; /* Use flex to center the button */
}

.action-btn {
    background-color: rgba(255, 255, 255, 0.2); /* Semi-transparent white */
    border: none;
    border-radius: 50%;
    width: 40px;
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #ffffff;
    cursor: pointer;
    transition: background-color 0.3s;
    font-size: 18px; /* Icon size */
}

.action-btn:hover {
    background-color: #00ffff; /* Teal on hover */
    color: #1a1a1a;
}

.movie-details {
    padding: 15px;
    flex-grow: 1; /* Allow details to take up remaining space */
    display: flex;
    flex-direction: column;
}

.movie-details h3 {
    margin-bottom: 5px;
    font-size: 16px;
    white-space: normal; /* Allow title to wrap */
    overflow: hidden;
    text-overflow: ellipsis;
}

.movie-info {
    color: #888;
    font-size: 14px;
    margin-bottom: 10px;
}

.rating {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-top: auto; /* Push rating to the bottom */
}

.stars {
    color: #ffd700; /* Gold color for stars */
    font-size: 1em;
}
.stars i {
    margin-right: 2px; /* Small spacing between stars */
}

.rating-count {
    color: #888;
    font-size: 14px;
}

/* Empty State Styles */
.empty-state {
    grid-column: 1 / -1; /* Span across all columns */
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 40px;
    text-align: center;
    background-color: #242424;
    border-radius: 15px;
    margin: 20px 0;
    color: #ffffff;
}

.empty-state i {
    font-size: 4em;
    color: #00ffff;
    margin-bottom: 20px;
}

.empty-state p {
    margin: 5px 0;
    font-size: 1.2em;
}

.empty-state .subtitle {
    color: #666;
    font-size: 0.9em;
}

.empty-state .subtitle a {
     color: #00ffff;
     text-decoration: none;
     font-weight: bold;
}
.empty-state .subtitle a:hover {
     text-decoration: underline;
}

/* Full width empty state for initial display */
.empty-state.full-width {
    width: 100%;
    margin: 0; /* Remove extra margin if it's the only content */
    min-height: 300px; /* Give it some height */
}

/* Alert Messages */
.alert {
    padding: 10px 20px;
    margin-bottom: 20px;
    border-radius: 8px;
    font-size: 14px;
    text-align: center;
}

.alert.success {
    background-color: #00ff0033; /* Green transparent */
    color: #00ff00; /* Green text */
    border: 1px solid #00ff0088;
}

.alert.error {
    background-color: #ff000033; /* Red transparent */
    color: #ff0000; /* Red text */
    border: 1px solid #ff000088;
}

/* Scrollbar Styles (Copy from beranda/styles.css) */
.main-content::-webkit-scrollbar {
    width: 8px;
}

.main-content::-webkit-scrollbar-track {
    background: #242424;
}

.main-content::-webkit-scrollbar-thumb {
    background: #363636;
    border-radius: 4px;
}

.main-content::-webkit-scrollbar-thumb:hover {
    background: #00ffff;
}

```

**12. `review/index.php` (Modified)**

Fetch all movies from the database, use them in the grid, and link to the details page. Update navigation.

```php
<?php
// review/index.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated (optional, but good practice for review section)
redirectIfNotAuthenticated();

// Fetch all movies from the database
$movies = getAllMovies(); // This function now fetches average_rating and genres

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Review</title>
    <link rel="stylesheet" href="styles.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Using FA 6 -->
</head>
<body>
    <div class="container">
        <nav class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li>
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favourites</span></a></li>
                <li class="active"><a href="#"><i class="fas fa-star"></i> <span>Review</span></a></li>
                <li><a href="../manage/indeks.php"><i class="fas fa-film"></i> <span>Manage</span></a></li>
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Corrected logout link -->
                </ul>
            </div>
        </nav>
        <main class="main-content">
            <div class="review-header">
                <h1>Movie Reviews</h1>
                <div class="search-bar">
                    <input type="text" id="searchInput" placeholder="Search movies...">
                    <button><i class="fas fa-search"></i></button>
                </div>
            </div>
            <div class="review-grid">
                <?php if (!empty($movies)): ?>
                    <?php foreach ($movies as $movie): ?>
                        <div class="movie-card" onclick="window.location.href='movie-details.php?id=<?php echo $movie['movie_id']; ?>'"> <!-- Link to movie details page -->
                            <div class="movie-poster">
                                <img src="<?php echo htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                <div class="movie-actions">
                                     <!-- Favorite Button (will be handled on details page) -->
                                     <!-- <button class="action-btn"><i class="fas fa-heart"></i></button> -->
                                </div>
                            </div>
                            <div class="movie-details">
                                <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                                <p class="movie-info"><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?></p>
                                <div class="rating">
                                    <div class="stars">
                                         <?php
                                        // Display average rating stars
                                        $average_rating = floatval($movie['average_rating']);
                                        for ($i = 1; $i <= 5; $i++) {
                                            if ($i <= $average_rating) {
                                                echo '<i class="fas fa-star"></i>'; // Full star
                                            } else if ($i - 0.5 <= $average_rating) {
                                                echo '<i class="fas fa-star-half-alt"></i>'; // Half star
                                            } else {
                                                echo '<i class="far fa-star"></i>'; // Empty star
                                            }
                                        }
                                        ?>
                                    </div>
                                     <span class="rating-count">(<?php echo htmlspecialchars($movie['average_rating']); ?>)</span>
                                </div>
                            </div>
                        </div>
                    <?php endforeach; ?>
                <?php else: ?>
                     <div class="empty-state full-width">
                         <i class="fas fa-film"></i>
                         <p>No movies available to review yet.</p>
                         <p class="subtitle">Check back later or <a href="../manage/upload.php">Upload a movie</a> if you are an admin.</p>
                     </div>
                <?php endif; ?>
            </div>
        </main>
    </div>

    <!-- Removed the movie details popup HTML -->
    <!-- The movie details will be on a separate page -->


    <script>
        document.addEventListener('DOMContentLoaded', function() {
            const searchInput = document.getElementById('searchInput');
            const movieCards = document.querySelectorAll('.review-grid .movie-card');
            const reviewGrid = document.querySelector('.review-grid');
             const initialEmptyState = document.querySelector('.empty-state.full-width');


            searchInput.addEventListener('input', function() {
                const searchTerm = this.value.toLowerCase();
                let visibleCardCount = 0;

                movieCards.forEach(card => {
                    const title = card.querySelector('h3').textContent.toLowerCase();
                    const info = card.querySelector('.movie-info').textContent.toLowerCase(); // Search genres/year too

                    if (title.includes(searchTerm) || info.includes(searchTerm)) {
                        card.style.display = '';
                         visibleCardCount++;
                    } else {
                        card.style.display = 'none';
                    }
                });

                 // Handle empty state visibility
                const searchEmptyState = document.querySelector('.search-empty-state');

                if (visibleCardCount === 0 && searchTerm !== '') {
                    if (!searchEmptyState) {
                         // Hide the initial empty state if it exists and we are searching
                        if(initialEmptyState) initialEmptyState.style.display = 'none';

                        const emptyState = document.createElement('div');
                        emptyState.className = 'empty-state search-empty-state full-width'; // Add full-width class
                        emptyState.innerHTML = `
                            <i class="fas fa-search"></i>
                            <p>No movies found matching "${htmlspecialchars(searchTerm)}"</p>
                            <p class="subtitle">Try a different search term</p>
                        `;
                        reviewGrid.appendChild(emptyState);
                    } else {
                         // Update text if search empty state already exists
                         searchEmptyState.querySelector('p:first-of-type').innerText = `No movies found matching "${htmlspecialchars(searchTerm)}"`;
                         searchEmptyState.style.display = 'flex'; // Ensure it's visible
                    }
                } else {
                    // Remove search empty state if cards are visible or search is cleared
                    if (searchEmptyState) {
                        searchEmptyState.remove();
                    }
                     // Show initial empty state if no movies were loaded AND search is cleared
                    if (movieCards.length === 0 && searchTerm === '' && initialEmptyState) {
                         initialEmptyState.style.display = 'flex';
                    }
                }
            });

            // Trigger search when search button is clicked
            const searchButton = document.querySelector('.search-bar button');
            searchButton.addEventListener('click', function() {
                const event = new Event('input');
                searchInput.dispatchEvent(event);
            });
        });

         // Helper function for HTML escaping (client-side)
         function htmlspecialchars(str) {
             if (typeof str !== 'string') return str;
             return str.replace(/&/g, '&amp;')
                       .replace(/</g, '&lt;')
                       .replace(/>/g, '&gt;')
                       .replace(/"/g, '&quot;')
                       .replace(/'/g, '&#039;');
         }
    </script>
</body>
</html>
```

**13. `review/movie-details.php` (Modified)**

Fetch movie details and comments based on URL parameter, implement rating and commenting, and favorite toggle.

```php
<?php
// review/movie-details.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated
redirectIfNotAuthenticated();

// Get authenticated user ID
$userId = $_SESSION['user_id'];
$user = getAuthenticatedUser(); // Fetch user details for comments

// Get movie ID from URL parameter
$movieId = filter_var($_GET['id'] ?? null, FILTER_VALIDATE_INT);

// Fetch movie details from the database
$movie = null;
if ($movieId) {
    $movie = getMovieById($movieId);
}

// Handle movie not found
if (!$movie) {
    // Redirect to review page with an error message
    $_SESSION['error_message'] = 'Movie not found.';
    header('Location: index.php');
    exit;
}

// Fetch comments for the movie
$comments = getMovieReviews($movieId);

// Check if the movie is favorited by the current user
$isFavorited = isMovieFavorited($movieId, $userId);

// Handle comment submission
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['comment'])) {
    $commentText = trim($_POST['comment']);
    // Rating is not collected in the main comment input field, let's assume rating is handled separately or via the star rating UI
    // For now, let's assume a default rating or add a hidden input/JS logic to send rating with comment
    // Let's add a separate rating submission endpoint or include it with the comment form
    // For simplicity in this single file, let's assume the rating form and comment form are separate or combined client-side.
    // We'll add a dedicated rating system later. For now, comments are just text.
    // To submit a rating *with* a comment, we'd need an input for rating in the form.
    // Let's modify the comment form to include a rating input.

    // If using the simplified textarea comment, let's add a basic rating input
    // If you intend a separate rating submission, this comment processing needs adjustment.

    // --- Handling combined comment and rating submission ---
    $submittedRating = filter_var($_POST['rating'] ?? null, FILTER_VALIDATE_FLOAT);
    if ($submittedRating === false || $submittedRating < 0.5 || $submittedRating > 5) {
        // If rating is required with comment, add an error
        // For now, allow comments without rating or use a default rating if none provided
        // Let's add a default rating (e.g., 0 or previous rating) if not explicitly provided in the form.
        // Or better, make rating required for a review submission.
        // Let's enforce rating is sent with comment.
         if ($submittedRating === false || $submittedRating < 0.5 || $submittedRating > 5) {
              $_SESSION['error_message'] = 'Please provide a valid rating (0.5 to 5).';
              header("Location: movie-details.php?id={$movieId}");
              exit;
         }
    }

     if (!empty($commentText) || ($submittedRating !== false && $submittedRating >= 0.5 && $submittedRating <= 5)) {
         // If comment is empty, maybe save just the rating? The createReview function supports comment TEXT, so empty is fine.
         // If rating is not sent, let's use a default or reject?
         // Let's require rating with comment submission for a 'review'.
         if ($submittedRating !== false && $submittedRating >= 0.5 && $submittedRating <= 5) {
              if (createReview($movieId, $userId, $submittedRating, $commentText)) {
                  $_SESSION['success_message'] = 'Your review has been submitted!';
              } else {
                  $_SESSION['error_message'] = 'Failed to submit your review.';
              }
         } else {
              $_SESSION['error_message'] = 'Rating is required with a comment.';
         }
     } else {
         $_SESSION['error_message'] = 'Comment cannot be empty.';
     }

    header("Location: movie-details.php?id={$movieId}"); // Redirect after processing
    exit;
}

// Handle Favorite/Unfavorite action (using GET for simplicity, POST recommended for production)
// A dedicated AJAX endpoint or a form would be more robust
if (isset($_GET['action']) && ($GET['action'] === 'favorite' || $_GET['action'] === 'unfavorite')) {
    $action = $_GET['action'];
    $targetMovieId = filter_var($_GET['movie_id'] ?? null, FILTER_VALIDATE_INT);

    if ($targetMovieId && $targetMovieId === $movieId) { // Ensure action is for the current movie
        if ($action === 'favorite') {
            if (addToFavorites($targetMovieId, $userId)) {
                 $_SESSION['success_message'] = 'Movie added to favorites!';
            } else {
                 $_SESSION['error_message'] = 'Failed to add movie to favorites (maybe already added?).';
            }
        } elseif ($action === 'unfavorite') {
             if (removeFromFavorites($targetMovieId, $userId)) {
                 $_SESSION['success_message'] = 'Movie removed from favorites!';
            } else {
                 $_SESSION['error_message'] = 'Failed to remove movie from favorites.';
            }
        }
    } else {
         $_SESSION['error_message'] = 'Invalid action or movie ID for favorite.';
    }
     // Redirect back to the movie details page
    header("Location: movie-details.php?id={$movieId}");
    exit;
}


// Get messages from session
$success_message = isset($_SESSION['success_message']) ? $_SESSION['success_message'] : null;
unset($_SESSION['success_message']);
$error_message = isset($_SESSION['error_message']) ? $_SESSION['error_message'] : null;
unset($_SESSION['error_message']);

// Determine poster image source
$posterSrc = htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg');

// Determine trailer source
$trailerUrl = null;
if (!empty($movie['trailer_url'])) {
    // Assume YouTube URL and extract video ID
    parse_str( parse_url( $movie['trailer_url'], PHP_URL_QUERY ), $vars );
    $youtubeVideoId = $vars['v'] ?? null;
    if ($youtubeVideoId) {
        $trailerUrl = "https://www.youtube.com/embed/{$youtubeVideoId}";
    } else {
         // Handle other video URL types if needed
         $trailerUrl = htmlspecialchars($movie['trailer_url']);
    }

} elseif (!empty($movie['trailer_file'])) {
    // Assume local file path
    $trailerUrl = htmlspecialchars('../uploads/trailers/' . $movie['trailer_file']); // Adjust path if necessary
}

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?php echo htmlspecialchars($movie['title']); ?> - RATE-TALES</title>
    <link rel="stylesheet" href="styles.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Using FA 6 -->
    <style>
        /* Specific styles for the movie details page */
        .movie-details-page {
            padding: 2rem;
            color: white;
        }

        .back-button {
            display: inline-flex;
            align-items: center;
            gap: 0.5rem;
            color: #00ffff; /* Teal color for back button */
            text-decoration: none;
            margin-bottom: 2rem;
            font-size: 1.1rem; /* Slightly smaller font */
            transition: color 0.3s;
        }
         .back-button:hover {
             color: #00cccc;
         }

        .movie-header {
            display: flex;
            gap: 2rem;
            margin-bottom: 2rem;
            flex-wrap: wrap; /* Allow wrapping on smaller screens */
        }

        .movie-poster-large {
            width: 300px;
            height: 450px;
            border-radius: 15px;
            overflow: hidden;
            box-shadow: 0 5px 15px rgba(0,0,0,0.3); /* Add shadow */
             flex-shrink: 0; /* Prevent shrinking */
             margin: auto; /* Center if wraps */
        }
         @media (max-width: 768px) {
             .movie-poster-large {
                 width: 200px;
                 height: 300px;
             }
         }


        .movie-poster-large img {
            width: 100%;
            height: 100%;
            object-fit: cover;
        }

        .movie-info-large {
            flex: 1;
            min-width: 300px; /* Ensure text area doesn't get too small */
        }

        .movie-title-large {
            font-size: 2.8rem; /* Slightly smaller title */
            margin-bottom: 1rem;
             color: #00ffff; /* Teal title */
        }

        .movie-meta {
            color: #888;
            margin-bottom: 1.5rem;
            font-size: 1.1rem;
        }

        .rating-large {
            display: flex;
            align-items: center;
            gap: 1rem;
            margin-bottom: 1.5rem;
        }

        .rating-large .stars {
            color: #ffd700; /* Gold stars */
            font-size: 1.8rem; /* Larger stars */
        }
         .rating-large .stars i {
             margin-right: 3px;
         }

        .movie-description {
            line-height: 1.8;
            margin-bottom: 2rem;
             color: #ccc; /* Lighter text for description */
        }

        .action-buttons {
            display: flex;
            gap: 1rem;
            flex-wrap: wrap; /* Allow buttons to wrap */
        }

        .action-button {
            padding: 1rem 2rem;
            border: none;
            border-radius: 10px;
            font-size: 1.1rem;
            cursor: pointer;
            display: flex;
            align-items: center;
            gap: 0.8rem; /* Increased gap */
            transition: all 0.3s ease;
            font-weight: bold; /* Make button text bold */
        }

        .watch-trailer {
            background-color: #e50914; /* Netflix red */
            color: white;
        }

        .add-favorite {
            background-color: #333; /* Dark gray */
            color: white;
        }
         .add-favorite.favorited {
             background-color: #00ffff; /* Teal when favorited */
             color: #1a1a1a; /* Dark text when favorited */
         }


        .action-button:hover {
            transform: translateY(-3px); /* More pronounced lift */
            opacity: 0.9;
        }

        .comments-section {
            margin-top: 3rem;
             background-color: #242424; /* Section background */
             padding: 20px;
             border-radius: 15px;
        }

        .comments-header {
            font-size: 1.8rem; /* Larger header */
            margin-bottom: 1.5rem;
             color: #00ffff; /* Teal header */
             border-bottom: 1px solid #333; /* Separator line */
             padding-bottom: 15px;
        }

        .comment-input-area {
            margin-bottom: 2rem;
             padding: 15px;
             background-color: #1a1a1a; /* Input area background */
             border-radius: 10px;
        }
         .comment-input-area h3 {
             font-size: 1.2rem;
             margin-bottom: 1rem;
             color: #ccc;
         }

         .rating-input-stars {
             display: flex;
             align-items: center;
             gap: 5px; /* Space between rating stars */
             margin-bottom: 1rem;
         }
         .rating-input-stars i {
             font-size: 1.5rem;
             color: #888; /* Default gray color */
             cursor: pointer;
             transition: color 0.2s, transform 0.2s;
         }
          .rating-input-stars i:hover,
          .rating-input-stars i.hovered,
          .rating-input-stars i.rated {
              color: #ffd700; /* Gold on hover/rated */
              transform: scale(1.1); /* Slightly enlarge */
          }
         .rating-input-stars input[type="hidden"] {
             display: none; /* Hide the hidden input */
         }


        .comment-input {
            width: 100%;
            padding: 1rem;
            border: none;
            border-radius: 10px;
            background-color: #333;
            color: white;
            margin-bottom: 1rem;
            resize: vertical;
             min-height: 80px; /* Minimum height */
        }
         .comment-input:focus {
              outline: none;
              border: 1px solid #00ffff;
              box-shadow: 0 0 0 2px rgba(0, 255, 255, 0.2);
         }


        .comment-submit-btn {
            display: block; /* Make button block */
            width: 150px; /* Fixed width */
            margin-left: auto; /* Push to the right */
            padding: 10px 20px;
            border: none;
            border-radius: 8px;
            background-color: #00ffff;
            color: #1a1a1a;
            cursor: pointer;
            font-size: 1em;
            font-weight: bold;
            transition: background-color 0.3s, transform 0.3s;
        }
        .comment-submit-btn:hover {
            background-color: #00cccc;
             transform: translateY(-2px);
        }


        .comment-list {
            display: flex;
            flex-direction: column;
            gap: 1rem;
        }

        .comment {
            background-color: #1a1a1a; /* Darker background for individual comments */
            padding: 1rem;
            border-radius: 10px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2); /* Subtle shadow */
        }

        .comment-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 0.5rem;
            font-size: 0.9em; /* Smaller font for header */
             color: #b0e0e6; /* Light gray color */
        }

        .comment-header strong {
             color: #00ffff; /* Teal color for username */
             font-weight: bold;
             margin-right: 10px;
        }

         .comment-rating-display {
             display: flex;
             align-items: center;
             gap: 5px;
             margin-right: 10px;
         }
         .comment-rating-display .stars {
             color: #ffd700; /* Gold stars */
             font-size: 0.9em;
         }
         .comment-rating-display span {
             font-size: 0.9em;
             color: #b0e0e6;
         }


        .comment-actions {
            display: flex;
            gap: 1rem;
            color: #888;
             font-size: 0.8em; /* Smaller font for actions */
        }

        .comment-actions i {
            cursor: pointer;
            transition: color 0.3s;
        }

        .comment-actions i:hover {
            color: #00ffff; /* Teal on hover */
        }

        .comment p {
            color: #ccc; /* Lighter text for comment content */
            line-height: 1.5;
        }

        /* Modal styles (Trailer) - Copy from review/styles.css */
        .trailer-modal {
            display: none;
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0, 0, 0, 0.9);
            z-index: 1000;
            justify-content: center;
            align-items: center;
        }

        .trailer-modal.active {
            display: flex;
        }

        .trailer-content {
            width: 90%; /* Make it wider */
            max-width: 1000px; /* Max width */
            position: relative;
        }

        .close-trailer {
            position: absolute;
            top: -40px; /* Position above the video */
            right: 0;
            color: white;
            font-size: 2.5rem; /* Larger close icon */
            cursor: pointer;
            transition: color 0.3s;
        }
         .close-trailer:hover {
             color: #ccc;
         }

        .video-container {
            position: relative;
            padding-bottom: 56.25%; /* 16:9 Aspect Ratio */
            height: 0;
            overflow: hidden;
             background-color: black; /* Ensure black background behind video */
        }

        .video-container iframe {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            border: none;
        }

         /* Alert Messages (Copy from favorite/styles.css) */
        .alert {
            padding: 10px 20px;
            margin-bottom: 20px;
            border-radius: 8px;
            font-size: 14px;
            text-align: center;
        }

        .alert.success {
            background-color: #00ff0033;
            color: #00ff00;
            border: 1px solid #00ff0088;
        }

        .alert.error {
            background-color: #ff000033;
            color: #ff0000;
            border: 1px solid #ff000088;
        }

        /* Scrollbar Styles (Copy from review/styles.css) */
        .main-content::-webkit-scrollbar {
            width: 8px;
        }

        .main-content::-webkit-scrollbar-track {
            background: #242424;
        }

        .main-content::-webkit-scrollbar-thumb {
            background: #363636;
            border-radius: 4px;
        }

        .main-content::-webkit-scrollbar-thumb:hover {
            background: #00ffff;
        }


    </style>
</head>
<body>
    <div class="container">
        <nav class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li>
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favourites</span></a></li>
                <li class="active"><a href="index.php"><i class="fas fa-star"></i> <span>Review</span></a></li>
                <li><a href="../manage/indeks.php"><i class="fas fa-film"></i> <span>Manage</span></a></li>
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Corrected logout link -->
                </ul>
            </div>
        </nav>
        <main class="main-content">
            <div class="movie-details-page">
                <a href="index.php" class="back-button">
                    <i class="fas fa-arrow-left"></i>
                    <span>Back to Reviews</span>
                </a>

                 <?php if ($success_message): ?>
                    <div class="alert success"><?php echo htmlspecialchars($success_message); ?></div>
                <?php endif; ?>
                <?php if ($error_message): ?>
                    <div class="alert error"><?php echo htmlspecialchars($error_message); ?></div>
                <?php endif; ?>


                <div class="movie-header">
                    <div class="movie-poster-large">
                        <img src="<?php echo $posterSrc; ?>" alt="<?php echo htmlspecialchars($movie['title']); ?> Poster">
                    </div>
                    <div class="movie-info-large">
                        <h1 class="movie-title-large"><?php echo htmlspecialchars($movie['title']); ?></h1>
                        <p class="movie-meta"><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?> | <?php echo htmlspecialchars($movie['age_rating']); ?> | <?php echo $movie['duration_hours'] > 0 ? htmlspecialchars($movie['duration_hours']) . 'h ' : ''; ?><?php echo htmlspecialchars($movie['duration_minutes']); ?>m</p>
                        <div class="rating-large">
                             <!-- Display average rating stars -->
                             <div class="stars">
                                <?php
                                $average_rating = floatval($movie['average_rating']);
                                for ($i = 1; $i <= 5; $i++) {
                                    if ($i <= $average_rating) {
                                        echo '<i class="fas fa-star"></i>'; // Full star
                                    } else if ($i - 0.5 <= $average_rating) {
                                        echo '<i class="fas fa-star-half-alt"></i>'; // Half star
                                    } else {
                                        echo '<i class="far fa-star"></i>'; // Empty star
                                    }
                                }
                                ?>
                            </div>
                            <span id="movie-rating"><?php echo htmlspecialchars($movie['average_rating']); ?>/5</span>
                        </div>
                        <p class="movie-description"><?php echo htmlspecialchars($movie['summary'] ?? 'No summary available.'); ?></p>
                        <div class="action-buttons">
                            <?php if ($trailerUrl): ?>
                                <button class="action-button watch-trailer" onclick="playTrailer('<?php echo $trailerUrl; ?>')">
                                    <i class="fas fa-play"></i>
                                    <span>Watch Trailer</span>
                                </button>
                            <?php endif; ?>

                             <!-- Favorite/Unfavorite button -->
                             <button class="action-button add-favorite <?php echo $isFavorited ? 'favorited' : ''; ?>"
                                     onclick="toggleFavorite(<?php echo $movie['movie_id']; ?>, <?php echo $isFavorited ? 'true' : 'false'; ?>)">
                                 <i class="fas fa-heart"></i>
                                 <span id="favorite-text"><?php echo $isFavorited ? 'Remove from Favorites' : 'Add to Favorites'; ?></span>
                             </button>
                        </div>
                    </div>
                </div>
                <div class="comments-section">
                    <h2 class="comments-header">Comments & Reviews</h2>

                     <div class="comment-input-area">
                         <h3>Leave a Review</h3>
                         <form action="movie-details.php?id=<?php echo $movie['movie_id']; ?>" method="POST">
                             <div class="rating-input-stars" id="rating-input-stars">
                                 <i class="far fa-star" data-rating="1"></i>
                                 <i class="far fa-star" data-rating="2"></i>
                                 <i class="far fa-star" data-rating="3"></i>
                                 <i class="far fa-star" data-rating="4"></i>
                                 <i class="far fa-star" data-rating="5"></i>
                                 <input type="hidden" name="rating" id="user-rating" value="0"> <!-- Hidden input for selected rating -->
                             </div>
                             <textarea class="comment-input" name="comment" placeholder="Write your comment or review here..."></textarea>
                             <button type="submit" class="comment-submit-btn">Submit Review</button>
                         </form>
                     </div>


                    <div class="comment-list">
                        <?php if (!empty($comments)): ?>
                            <?php foreach ($comments as $comment): ?>
                                <div class="comment">
                                    <div class="comment-header">
                                         <div style="display: flex; align-items: center;">
                                             <img src="<?php echo htmlspecialchars($comment['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($comment['username']) . '&background=random'); ?>" alt="Avatar" style="width: 25px; height: 25px; border-radius: 50%; margin-right: 10px; object-fit: cover;">
                                             <strong><?php echo htmlspecialchars($comment['username']); ?></strong>
                                         </div>
                                         <div style="display: flex; align-items: center; gap: 15px;">
                                            <div class="comment-rating-display">
                                                 <div class="stars">
                                                     <?php
                                                     $comment_rating = floatval($comment['rating']);
                                                     for ($i = 1; $i <= 5; $i++) {
                                                         if ($i <= $comment_rating) {
                                                             echo '<i class="fas fa-star"></i>';
                                                         } else if ($i - 0.5 <= $comment_rating) {
                                                             echo '<i class="fas fa-star-half-alt"></i>';
                                                         } else {
                                                             echo '<i class="far fa-star"></i>';
                                                         }
                                                     }
                                                     ?>
                                                 </div>
                                                <span>(<?php echo htmlspecialchars(number_format($comment_rating, 1)); ?>/5)</span>
                                            </div>
                                            <div class="comment-actions">
                                                <!-- Basic Placeholder Actions (Like/Dislike/Reply) -->
                                                <i class="fas fa-thumbs-up" title="Like"></i>
                                                <i class="fas fa-thumbs-down" title="Dislike"></i>
                                                <i class="fas fa-reply" title="Reply"></i>
                                            </div>
                                         </div>
                                    </div>
                                    <p><?php echo nl2br(htmlspecialchars($comment['comment'] ?? '')); ?></p> <!-- Use nl2br for line breaks -->
                                </div>
                            <?php endforeach; ?>
                         <?php else: ?>
                             <div class="empty-state" style="background-color: #1a1a1a; padding: 20px; border-radius: 10px;">
                                 <i class="fas fa-comment-dots"></i>
                                 <p>No comments yet. Be the first to review!</p>
                             </div>
                        <?php endif; ?>
                    </div>
                </div>
            </div>
        </main>
    </div>

    <!-- Trailer Modal -->
    <div id="trailer-modal" class="trailer-modal">
        <div class="trailer-content">
            <span class="close-trailer" onclick="closeTrailer()">&times;</span>
            <div class="video-container">
                <iframe id="trailer-iframe" src="" frameborder="0" allowfullscreen></iframe>
            </div>
        </div>
    </div>

    <script>
    // JavaScript for rating input
    const ratingStars = document.querySelectorAll('#rating-input-stars .fa-star, #rating-input-stars .fa-star-half-alt, #rating-input-stars .far.fa-star'); // Select all stars in input area
    const userRatingInput = document.getElementById('user-rating');
    let currentRating = 0; // Store the selected rating

    ratingStars.forEach(star => {
        star.addEventListener('mouseover', function() {
            highlightStars(this.getAttribute('data-rating'));
        });

        star.addEventListener('mouseout', function() {
             // Revert to the currently selected rating instead of clearing
            highlightStars(currentRating);
        });

        star.addEventListener('click', function() {
            currentRating = this.getAttribute('data-rating'); // Update selected rating
            userRatingInput.value = currentRating; // Set hidden input value
            highlightStars(currentRating, true); // Highlight and mark as rated
        });
    });

    function highlightStars(rating, isClick = false) {
        ratingStars.forEach((star, index) => {
            const starRating = parseInt(star.getAttribute('data-rating'));
             star.classList.remove('hovered', 'rated'); // Remove previous states

             if (starRating <= rating) {
                 star.classList.add(isClick ? 'rated' : 'hovered'); // Add rated or hovered class
                 star.classList.remove('far');
                 star.classList.add('fas');
             } else {
                 star.classList.remove('fas');
                 star.classList.add('far');
             }
        });
    }

    // Initial state based on a previously saved rating (if any)
    // You would fetch the user's existing rating from the DB on page load
    // and call highlightStars(fetchedRating, true)


    // JavaScript for trailer modal
    const trailerModal = document.getElementById('trailer-modal');
    const trailerIframe = document.getElementById('trailer-iframe');

    function playTrailer(videoSrc) {
        if (videoSrc) {
            trailerIframe.src = videoSrc;
            trailerModal.classList.add('active');
        } else {
            alert('Trailer not available.');
        }
    }

    function closeTrailer() {
        trailerIframe.src = ''; // Stop the video by clearing the src
        trailerModal.classList.remove('active');
    }

    // Close modal when clicking outside the content
    trailerModal.addEventListener('click', function(e) {
        if (e.target === this || e.target.classList.contains('close-trailer')) {
            closeTrailer();
        }
    });


     // JavaScript for favorite toggle
     function toggleFavorite(movieId, isFavorited) {
         const favoriteButton = document.querySelector('.action-button.add-favorite');
         const favoriteText = document.getElementById('favorite-text');

         // Determine the action based on current state
         const action = isFavorited ? 'unfavorite' : 'favorite';

         // Create a temporary form to submit the action via POST
         // Using POST is more appropriate for state-changing actions than GET
         const form = document.createElement('form');
         form.method = 'GET'; // Use GET for simplicity with current PHP handler, but POST is better
         form.action = `movie-details.php?id=${movieId}`;

         const actionInput = document.createElement('input');
         actionInput.type = 'hidden';
         actionInput.name = 'action';
         actionInput.value = action;
         form.appendChild(actionInput);

         const movieIdInput = document.createElement('input');
         movieIdInput.type = 'hidden';
         movieIdInput.name = 'movie_id';
         movieIdInput.value = movieId;
         form.appendChild(movieIdInput);

         document.body.appendChild(form); // Append to body to submit
         form.submit(); // Submit the form
     }

     // Helper function for HTML escaping (client-side)
     function htmlspecialchars(str) {
         if (typeof str !== 'string') return str;
         return str.replace(/&/g, '&amp;')
                   .replace(/</g, '&lt;')
                   .replace(/>/g, '&gt;')
                   .replace(/"/g, '&quot;')
                   .replace(/'/g, '&#039;');
     }

    </script>
</body>
</html>
```

**14. `review/styles.css` (Modified)**

Apply consistent styling, including the sidebar and modal.

```css
/* review/styles.css */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

body {
    background-color: #1a1a1a; /* Dark background */
    color: #ffffff; /* White text */
}

.container {
    display: flex;
    min-height: 100vh;
}

/* Sidebar Styles (Copy from beranda/styles.css for consistency) */
.sidebar {
    width: 250px;
    background-color: #242424; /* Dark gray sidebar */
    padding: 20px;
    display: flex;
    flex-direction: column;
    position: fixed;
    height: 100vh;
    top: 0;
    left: 0;
    z-index: 100;
    box-shadow: 2px 0 5px rgba(0,0,0,0.3);
}

.logo h2 {
    color: #00ffff; /* Teal logo color */
    margin-bottom: 40px;
    font-size: 24px;
     text-align: center;
}

.nav-links {
    list-style: none;
    flex-grow: 1;
     padding-top: 20px;
}

.nav-links li, .bottom-links li {
    margin-bottom: 15px;
}

.nav-links a, .bottom-links a {
    color: #ffffff;
    text-decoration: none;
    display: flex;
    align-items: center;
    padding: 12px 15px;
    border-radius: 8px;
    transition: background-color 0.3s, color 0.3s;
    font-size: 15px;
}

.nav-links a:hover, .bottom-links a:hover {
    background-color: #363636;
    color: #00ffff;
}

.nav-links i, .bottom-links i {
    margin-right: 15px;
    width: 20px;
     text-align: center;
}

.active a {
    background-color: #363636;
    color: #00ffff;
    font-weight: bold;
}
.active a i {
    color: #00ffff;
}


.bottom-links {
    margin-top: auto;
    list-style: none;
    padding-top: 20px;
    border-top: 1px solid #363636;
}

/* Main Content Styles */
.main-content {
    flex-grow: 1;
    margin-left: 250px; /* Match sidebar width */
    padding: 20px;
    max-height: 100vh;
    overflow-y: auto; /* Enable vertical scrolling */
}

/* Review Header Styles */
.review-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 30px;
    padding: 20px;
    background-color: #242424; /* Dark gray background */
    border-radius: 15px;
}

.review-header h1 {
    font-size: 28px;
    color: #00ffff; /* Teal header */
}

.search-bar {
    display: flex;
    gap: 10px;
}

.search-bar input {
    padding: 10px 15px;
    border-radius: 8px;
    border: none;
    background-color: #363636; /* Darker input background */
    color: #ffffff;
    width: 300px;
     transition: border-color 0.3s, box-shadow 0.3s;
}
.search-bar input:focus {
     border-color: #00ffff;
     box-shadow: 0 0 0 2px rgba(0, 255, 255, 0.2);
     outline: none;
}


.search-bar button {
    padding: 10px 20px;
    border-radius: 8px;
    border: none;
    background-color: #00ffff; /* Teal button */
    color: #1a1a1a; /* Dark text */
    cursor: pointer;
    transition: background-color 0.3s;
}

.search-bar button:hover {
    background-color: #00cccc; /* Darker teal on hover */
}

/* Review Grid Styles */
.review-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 20px;
    padding: 20px 0; /* Remove left/right padding to align with main content */
}

.movie-card {
    background-color: #242424;
    border-radius: 15px;
    overflow: hidden;
    transition: transform 0.3s;
    display: flex;
    flex-direction: column;
    position: relative;
}

.movie-card:hover {
    transform: translateY(-5px);
}

.movie-poster {
    position: relative;
    width: 100%;
    padding-top: 150%; /* Aspect ratio 2:3 */
    flex-shrink: 0;
    cursor: pointer;
}

.movie-poster img {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.movie-actions {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    padding: 15px;
    background: linear-gradient(transparent, rgba(0, 0, 0, 0.8));
    display: flex;
    justify-content: center;
    gap: 15px;
    opacity: 0;
    transition: opacity 0.3s ease-in-out;
     z-index: 10;
}

.movie-card:hover .movie-actions {
    opacity: 1;
}

.action-btn {
    background-color: rgba(255, 255, 255, 0.2);
    border: none;
    border-radius: 50%;
    width: 40px;
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #ffffff;
    cursor: pointer;
    transition: background-color 0.3s;
    font-size: 18px; /* Icon size */
}

.action-btn:hover {
    background-color: #00ffff;
    color: #1a1a1a;
}

.movie-details {
    padding: 15px;
    flex-grow: 1;
    display: flex;
    flex-direction: column;
}

.movie-details h3 {
    font-size: 16px;
    margin-bottom: 5px;
     white-space: normal;
    overflow: hidden;
    text-overflow: ellipsis;
}

.movie-info {
    color: #888888;
    font-size: 14px;
    margin-bottom: 12px;
}

.rating {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-top: auto;
}

.stars {
    color: #ffd700; /* Gold color for stars */
    font-size: 1em;
}
.stars i {
    margin-right: 2px;
}

.rating-count {
    color: #888888;
    font-size: 14px;
}

/* Empty State Styles */
.empty-state {
    grid-column: 1 / -1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 40px;
    text-align: center;
    background-color: #242424;
    border-radius: 15px;
    margin: 20px 0;
    color: #ffffff;
}

.empty-state i {
    font-size: 4em;
    color: #00ffff;
    margin-bottom: 20px;
}

.empty-state p {
    margin: 5px 0;
    font-size: 1.2em;
}

.empty-state .subtitle {
    color: #666;
    font-size: 0.9em;
}

.empty-state .subtitle a {
     color: #00ffff;
     text-decoration: none;
     font-weight: bold;
}
.empty-state .subtitle a:hover {
     text-decoration: underline;
}

.empty-state.full-width {
    width: 100%;
    margin: 0;
    min-height: 300px;
}

/* Alert Messages (Copy from favorite/styles.css) */
.alert {
    padding: 10px 20px;
    margin-bottom: 20px;
    border-radius: 8px;
    font-size: 14px;
    text-align: center;
}

.alert.success {
    background-color: #00ff0033;
    color: #00ff00;
    border: 1px solid #00ff0088;
}

.alert.error {
    background-color: #ff000033;
    color: #ff0000;
    border: 1px solid #ff000088;
}


/* Scrollbar Styles (Copy from beranda/styles.css) */
.main-content::-webkit-scrollbar {
    width: 8px;
}

.main-content::-webkit-scrollbar-track {
    background: #242424;
}

.main-content::-webkit-scrollbar-thumb {
    background: #363636;
    border-radius: 4px;
}

.main-content::-webkit-scrollbar-thumb:hover {
    background: #00ffff;
}

/* --- Movie Details Popup Styles --- */
/* This section is kept but unused as we navigate to a new page */
/* Remove if desired, but keeping original styles for context */
.movie-details-popup {
    display: none; /* Hide the popup */
    /* Rest of styles are kept but not active */
    position: fixed;
    bottom: -100%;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.8);
    justify-content: center;
    align-items: flex-end;
    z-index: 1000;
    transition: bottom 0.3s ease-in-out;
}

.movie-details-popup.active {
    bottom: 0;
    display: flex; /* Show the popup when active */
}

.popup-content {
    background-color: #1a1a1a;
    width: 100%;
    max-width: 800px;
    padding: 2rem;
    border-radius: 20px 20px 0 0;
    color: white;
}

.popup-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1rem;
}

.popup-header h2 {
    font-size: 2rem;
    margin: 0;
}

.popup-rating {
    display: flex;
    align-items: center;
    gap: 0.5rem;
}

.popup-rating .rating-stars {
    display: flex;
    gap: 0.5rem;
    margin-right: 1rem;
}

.popup-rating .rating-stars i {
    color: #ffd700;
    cursor: pointer;
    font-size: 1.5rem;
    transition: transform 0.2s;
}

.popup-rating .rating-stars i:hover {
    transform: scale(1.2);
}

#popup-year-genre {
    color: #888;
    margin-bottom: 1rem;
}

#popup-description {
    line-height: 1.6;
    margin-bottom: 2rem;
}

.popup-actions {
    display: flex;
    gap: 1rem;
}

.popup-actions button {
    padding: 0.8rem 1.5rem;
    border: none;
    border-radius: 10px;
    background-color: #333;
    color: white;
    cursor: pointer;
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 1rem;
}

.popup-actions .watch-now-btn {
    background-color: #e50914;
}

.popup-actions .read-more-btn {
    background-color: #2c3e50;
}

.popup-actions button:hover {
    transform: translateY(-2px);
    opacity: 0.9;
}
```

**15. `manage/indeks.php` (Modified)**

Fetch user's uploaded movies. Implement delete action. Update navigation.

```php
<?php
// manage/indeks.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated
redirectIfNotAuthenticated();

// Get authenticated user ID
$userId = $_SESSION['user_id'];

// Fetch movies uploaded by the current user
$movies = getUserUploadedMovies($userId);

// Handle delete movie action
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['delete_movie_id'])) {
    $movieIdToDelete = filter_var($_POST['delete_movie_id'], FILTER_VALIDATE_INT);

    if ($movieIdToDelete) {
        try {
            // Start a transaction because deleting a movie might involve deleting related data
            $pdo->beginTransaction();

            // Delete related data first (reviews, favorites, genres)
            // Due to ON DELETE CASCADE in schema, this might not be explicitly needed for movie_genres,
            // reviews, and favorites if FK constraints are set up correctly.
            // But explicit deletion can be safer or required depending on FK setup.
            // Assuming CASCADE is correctly set, deleting the movie is enough.
            // Double-check your schema!
            $stmt = $pdo->prepare("DELETE FROM movies WHERE movie_id = ? AND uploaded_by = ?"); // Ensure only the uploader can delete
            $stmt->execute([$movieIdToDelete, $userId]);

            // Check if a row was actually deleted
            if ($stmt->rowCount() > 0) {
                 // Optional: Delete associated files (poster, trailer) from the server
                 // This is more complex as you need the file paths before deleting the DB record.
                 // You would fetch the movie first, get the paths, then delete the record, then delete files.
                 // For simplicity here, file deletion is omitted.

                $pdo->commit();
                $_SESSION['success_message'] = 'Movie deleted successfully.';
            } else {
                 // Movie not found OR not uploaded by the current user
                $pdo->rollBack();
                $_SESSION['error_message'] = 'Movie not found or you do not have permission to delete it.';
            }

        } catch (PDOException $e) {
            $pdo->rollBack();
            error_log("Database error during movie deletion: " . $e->getMessage());
            $_SESSION['error_message'] = 'An internal error occurred while deleting the movie.';
        }
    } else {
        $_SESSION['error_message'] = 'Invalid movie ID.';
    }

    header('Location: indeks.php'); // Redirect to prevent form resubmission
    exit;
}


// Get messages from session
$success_message = isset($_SESSION['success_message']) ? $_SESSION['success_message'] : null;
unset($_SESSION['success_message']);
$error_message = isset($_SESSION['error_message']) ? $_SESSION['error_message'] : null;
unset($_SESSION['error_message']);

?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Manage Movies - RatingTales</title>
    <link rel="stylesheet" href="styles.css">
     <link rel="stylesheet" href="../review/styles.css"> <!-- Include review styles for movie card look -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Using FA 6 -->
</head>
<body>
    <div class="container">
        <!-- Sidebar -->
        <div class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li>
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favourites</span></a></li>
                <li><a href="../review/index.php"><i class="fas fa-star"></i> <span>Review</span></a></li>
                <li class="active"><a href="indeks.php"><i class="fas fa-film"></i> <span>Manage</span></a></li>
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
            </ul>
            <ul class="bottom-links">
                <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Corrected logout link -->
            </ul>
        </div>

        <!-- Main Content -->
        <main class="main-content">
            <div class="header">
                 <h1>Manage Movies</h1> <!-- Added header title -->
                <div class="search-bar">
                    <i class="fas fa-search"></i>
                    <input type="text" id="searchInput" placeholder="Search movies...">
                </div>
            </div>

             <?php if ($success_message): ?>
                <div class="alert success"><?php echo htmlspecialchars($success_message); ?></div>
            <?php endif; ?>
            <?php if ($error_message): ?>
                <div class="alert error"><?php echo htmlspecialchars($error_message); ?></div>
            <?php endif; ?>


            <!-- Movies Grid -->
            <div class="movies-grid review-grid"> <!-- Use review-grid class for styling -->
                <?php if (empty($movies)): ?>
                    <div class="empty-state full-width"> <!-- Use full-width empty state -->
                        <i class="fas fa-film"></i>
                        <p>No movies uploaded yet</p>
                        <p class="subtitle">Start adding your movies by clicking the upload button below</p>
                    </div>
                <?php else: ?>
                     <?php foreach ($movies as $movie): ?>
                        <div class="movie-card">
                            <div class="movie-poster">
                                <img src="<?php echo htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                <div class="movie-actions">
                                     <!-- Edit Button (Link to edit page) -->
                                     <a href="edit.php?id=<?php echo $movie['movie_id']; ?>" class="action-btn" title="Edit Movie">
                                         <i class="fas fa-edit"></i>
                                     </a>
                                     <!-- Delete Button (Form submission) -->
                                     <form action="indeks.php" method="POST" onsubmit="return confirm('Are you sure you want to delete this movie? This action cannot be undone.');">
                                         <input type="hidden" name="delete_movie_id" value="<?php echo $movie['movie_id']; ?>">
                                         <button type="submit" class="action-btn" title="Delete Movie"><i class="fas fa-trash"></i></button>
                                     </form>
                                </div>
                            </div>
                            <div class="movie-details">
                                <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                                <p class="movie-info"><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?></p>
                                <div class="rating">
                                    <div class="stars">
                                         <?php
                                        // Display average rating stars
                                        $average_rating = floatval($movie['average_rating']);
                                        for ($i = 1; $i <= 5; $i++) {
                                            if ($i <= $average_rating) {
                                                echo '<i class="fas fa-star"></i>';
                                            } else if ($i - 0.5 <= $average_rating) {
                                                echo '<i class="fas fa-star-half-alt"></i>';
                                            } else {
                                                echo '<i class="far fa-star"></i>';
                                            }
                                        }
                                        ?>
                                    </div>
                                    <span class="rating-count">(<?php echo htmlspecialchars($movie['average_rating']); ?>)</span>
                                </div>
                            </div>
                        </div>
                    <?php endforeach; ?>
                <?php endif; ?>
            </div>

            <!-- Action Buttons -->
            <div class="action-buttons">
                 <!-- Edit All button - currently not functional for multi-edit -->
                <!-- <button class="edit-all-btn" title="Edit All Movies">
                    <i class="fas fa-edit"></i>
                </button> -->
                <a href="upload.php" class="upload-btn" title="Upload New Movie">
                    <i class="fas fa-plus"></i>
                </a>
            </div>
        </main>
    </div>
    <script>
        document.addEventListener('DOMContentLoaded', function() {
            const searchInput = document.getElementById('searchInput');
            const movieCards = document.querySelectorAll('.movies-grid .movie-card');
            const moviesGrid = document.querySelector('.movies-grid');
            const initialEmptyState = document.querySelector('.empty-state.full-width');


            searchInput.addEventListener('input', function() {
                const searchTerm = this.value.toLowerCase();
                let visibleCardCount = 0;

                movieCards.forEach(card => {
                    const title = card.querySelector('h3').textContent.toLowerCase();
                    const info = card.querySelector('.movie-info').textContent.toLowerCase(); // Search genres/year too

                    if (title.includes(searchTerm) || info.includes(searchTerm)) {
                        card.style.display = '';
                         visibleCardCount++;
                    } else {
                        card.style.display = 'none';
                    }
                });

                // Handle empty state visibility
                const searchEmptyState = document.querySelector('.search-empty-state');

                if (visibleCardCount === 0 && searchTerm !== '') {
                    if (!searchEmptyState) {
                         // Hide the initial empty state if it exists and we are searching
                        if(initialEmptyState) initialEmptyState.style.display = 'none';

                        const emptyState = document.createElement('div');
                        emptyState.className = 'empty-state search-empty-state full-width'; // Add full-width class
                        emptyState.innerHTML = `
                            <i class="fas fa-search"></i>
                            <p>No uploaded movies found matching "${htmlspecialchars(searchTerm)}"</p>
                            <p class="subtitle">Try a different search term</p>
                        `;
                        moviesGrid.appendChild(emptyState);
                    } else {
                         // Update text if search empty state already exists
                         searchEmptyState.querySelector('p:first-of-type').innerText = `No uploaded movies found matching "${htmlspecialchars(searchTerm)}"`;
                         searchEmptyState.style.display = 'flex'; // Ensure it's visible
                    }
                } else {
                    // Remove search empty state if cards are visible or search is cleared
                    if (searchEmptyState) {
                        searchEmptyState.remove();
                    }
                     // Show initial empty state if no movies were loaded AND search is cleared
                    if (movieCards.length === 0 && searchTerm === '' && initialEmptyState) {
                         initialEmptyState.style.display = 'flex';
                    }
                }
            });
             // Trigger search when search button is clicked (if you add a button)
            // const searchButton = document.querySelector('.search-bar button');
            // if(searchButton) {
            //     searchButton.addEventListener('click', function() {
            //         const event = new Event('input');
            //         searchInput.dispatchEvent(event);
            //     });
            // }
        });

         // Helper function for HTML escaping (client-side)
         function htmlspecialchars(str) {
             if (typeof str !== 'string') return str;
             return str.replace(/&/g, '&amp;')
                       .replace(/</g, '&lt;')
                       .replace(/>/g, '> ') // Keep > as is for HTML structure
                       .replace(/"/g, '&quot;')
                       .replace(/'/g, '&#039;');
         }
    </script>
</body>
</html>
```

**16. `manage/styles.css` (Modified)**

Update styling for consistency, especially the sidebar and empty state.

```css
/* manage/styles.css */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

body {
    background-color: #1a1a1a;
    color: #ffffff;
}

.container {
    display: flex;
    min-height: 100vh;
}

/* Sidebar Styles (Copy from beranda/styles.css for consistency) */
.sidebar {
    width: 250px;
    background-color: #242424;
    padding: 20px;
    display: flex;
    flex-direction: column;
    position: fixed;
    height: 100vh;
    top: 0;
    left: 0;
    z-index: 100;
    box-shadow: 2px 0 5px rgba(0,0,0,0.3);
}

.logo h2 {
    color: #00ffff;
    margin-bottom: 40px;
    font-size: 24px;
     text-align: center;
}

.nav-links {
    list-style: none;
    flex-grow: 1;
     padding-top: 20px;
}

.nav-links li, .bottom-links li {
    margin-bottom: 15px;
}

.nav-links a, .bottom-links a {
    color: #ffffff;
    text-decoration: none;
    display: flex;
    align-items: center;
    padding: 12px 15px;
    border-radius: 8px;
    transition: background-color 0.3s, color 0.3s;
    font-size: 15px;
}

.nav-links a:hover, .bottom-links a:hover {
    background-color: #363636;
    color: #00ffff;
}

.nav-links i, .bottom-links i {
    margin-right: 15px;
    width: 20px;
     text-align: center;
}

.active a {
    background-color: #363636;
    color: #00ffff;
    font-weight: bold;
}
.active a i {
    color: #00ffff;
}


.bottom-links {
    margin-top: auto;
    list-style: none;
    padding-top: 20px;
    border-top: 1px solid #363636;
}

/* Main Content Styles */
.main-content {
    flex-grow: 1;
    margin-left: 250px;
    padding: 20px;
    max-height: 100vh;
    overflow-y: auto;
}

/* Header Styles */
.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 30px;
     padding: 0 20px; /* Add padding to match review-header */
}

.header h1 { /* Added style for header title */
    font-size: 28px;
    color: #00ffff; /* Teal header */
}


.search-bar {
    background-color: #242424;
    border-radius: 8px;
    padding: 10px 20px;
    display: flex;
    align-items: center;
    gap: 10px;
    width: 300px;
     transition: border-color 0.3s, box-shadow 0.3s;
}
.search-bar:focus-within { /* Style when input inside is focused */
     border: 1px solid #00ffff;
     box-shadow: 0 0 0 2px rgba(0, 255, 255, 0.2);
     outline: none;
}


.search-bar input {
    background: none;
    border: none;
    color: #fff;
    width: 100%;
    outline: none;
     font-size: 15px;
}

.search-bar input::placeholder {
    color: #666;
}


/* Movies Grid Styles (Used review-grid from review/styles.css) */
/* .movies-grid { ... removed ... } */

/* Empty State Styles (Copy from favorite/styles.css) */
.empty-state {
    grid-column: 1 / -1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 40px;
    text-align: center;
    background-color: #242424;
    border-radius: 15px;
    margin: 20px 0;
     color: #ffffff;
}

.empty-state i {
    font-size: 4em;
    color: #00ffff;
    margin-bottom: 20px;
}

.empty-state p {
    margin: 5px 0;
    font-size: 1.2em;
}

.empty-state .subtitle {
    color: #666;
    font-size: 0.9em;
}
.empty-state .subtitle a {
     color: #00ffff;
     text-decoration: none;
     font-weight: bold;
}
.empty-state .subtitle a:hover {
     text-decoration: underline;
}

.empty-state.full-width {
    width: 100%;
    margin: 0;
    min-height: 400px; /* Increased height */
}


/* Action Buttons Styles */
.action-buttons {
    position: fixed;
    bottom: 30px;
    right: 30px;
    display: flex;
    gap: 15px;
    z-index: 500; /* Ensure buttons are on top */
}

.action-buttons button,
.action-buttons a {
    width: 55px; /* Slightly larger buttons */
    height: 55px;
    border-radius: 50%;
    border: none;
    color: #fff;
    font-size: 1.3em; /* Larger icons */
    cursor: pointer;
    transition: transform 0.3s, background-color 0.3s, box-shadow 0.3s;
    display: flex;
    align-items: center;
    justify-content: center;
    text-decoration: none;
    box-shadow: 0 4px 10px rgba(0,0,0,0.3); /* Add shadow */
}

.upload-btn {
    background-color: #00FFFF; /* Teal upload button */
    color: #1a1a1a; /* Dark text */
}

.edit-all-btn {
    background-color: #363636; /* Dark gray edit button */
    color: #ffffff; /* White text */
}

.action-buttons button:hover,
.action-buttons a:hover {
    transform: scale(1.1);
    box-shadow: 0 6px 15px rgba(0,255,255,0.3); /* Teal glow on hover */
}

.upload-btn:hover {
    background-color: #00CCCC;
}

.edit-all-btn:hover {
    background-color: #444444;
     box-shadow: 0 6px 15px rgba(255,255,255,0.1); /* Subtle white glow */
}

/* Alert Messages (Copy from favorite/styles.css) */
.alert {
    padding: 10px 20px;
    margin-bottom: 20px;
    border-radius: 8px;
    font-size: 14px;
    text-align: center;
}

.alert.success {
    background-color: #00ff0033;
    color: #00ff00;
    border: 1px solid #00ff0088;
}

.alert.error {
    background-color: #ff000033;
    color: #ff0000;
    border: 1px solid #ff000088;
}

/* Scrollbar Styles (Copy from beranda/styles.css) */
.main-content::-webkit-scrollbar {
    width: 8px;
}

.main-content::-webkit-scrollbar-track {
    background: #242424;
}

.main-content::-webkit-scrollbar-thumb {
    background: #363636;
    border-radius: 4px;
}

.main-content::-webkit-scrollbar-thumb:hover {
    background: #00ffff;
}

```

**17. `manage/upload.php` (Modified)**

Implement file upload and database insertion logic. Update navigation.

```php
<?php
// manage/upload.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated
redirectIfNotAuthenticated();

// Get authenticated user ID
$userId = $_SESSION['user_id'];

$error_message = null;
$success_message = null;

// Handle form submission
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // 1. Collect and Sanitize Input
    $title = trim($_POST['movie-title'] ?? '');
    $summary = trim($_POST['movie-summary'] ?? '');
    $release_date = $_POST['release-date'] ?? '';
    $duration_hours = filter_var($_POST['duration-hours'] ?? 0, FILTER_VALIDATE_INT);
    $duration_minutes = filter_var($_POST['duration-minutes'] ?? 0, FILTER_VALIDATE_INT);
    $age_rating = $_POST['age-rating'] ?? '';
    $genres = $_POST['genre'] ?? []; // Array of selected genres
    $trailer_url = trim($_POST['trailer-link'] ?? '');
    // Trailer file will be handled via $_FILES

    $errors = [];

    // 2. Validate Input
    if (empty($title)) $errors[] = 'Movie Title is required.';
    if (empty($release_date)) $errors[] = 'Release Date is required.';
    if ($duration_hours === false || $duration_hours < 0) $errors[] = 'Valid Duration (Hours) is required.';
    if ($duration_minutes === false || $duration_minutes < 0 || $duration_minutes > 59) $errors[] = 'Valid Duration (Minutes) is required (0-59).';
    if (empty($age_rating)) $errors[] = 'Age Rating is required.';
    if (empty($genres)) $errors[] = 'At least one Genre must be selected.';

    // Validate genre values against allowed ENUM values (optional but good)
    $allowed_genres = ['action', 'adventure', 'comedy', 'drama', 'horror', 'supernatural', 'animation', 'sci-fi']; // Must match DB schema
    foreach ($genres as $genre) {
        if (!in_array($genre, $allowed_genres)) {
            $errors[] = 'Invalid genre selected.';
            break; // Stop checking if one invalid genre is found
        }
    }

    // 3. Handle File Uploads (Poster)
    $poster_image_path = null;
    // Check if a poster file was uploaded and if there's no error
    if (isset($_FILES['movie-poster']) && $_FILES['movie-poster']['error'] === UPLOAD_ERR_OK) {
        $posterFile = $_FILES['movie-poster'];
        $allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp']; // Allowed image types
        $maxFileSize = 5 * 1024 * 1024; // 5MB

        if (!in_array($posterFile['type'], $allowedTypes)) {
            $errors[] = 'Invalid poster file type. Only JPG, PNG, GIF, WEBP are allowed.';
        }
        if ($posterFile['size'] > $maxFileSize) {
            $errors[] = 'Poster file is too large. Maximum size is 5MB.';
        }

        // If no file errors yet, process the upload
        if (empty($errors)) {
            $fileExtension = pathinfo($posterFile['name'], PATHINFO_EXTENSION);
            $newFileName = uniqid('poster_', true) . '.' . $fileExtension;
            $destination = UPLOAD_DIR_POSTERS . $newFileName;

            if (move_uploaded_file($posterFile['tmp_name'], $destination)) {
                $poster_image_path = $newFileName; // Store just the filename/relative path
            } else {
                $errors[] = 'Failed to upload poster file.';
            }
        }
    } else {
         // If no poster file uploaded AND poster_image_path is empty (for edit), require it
        if (empty($poster_image_path)) { // This check is more relevant for editing, but keeping for simplicity
             // For upload, poster IS required based on form HTML.
             $errors[] = 'Movie Poster is required.';
        }
         // Handle specific upload errors
         if (isset($_FILES['movie-poster']) && $_FILES['movie-poster']['error'] !== UPLOAD_ERR_NO_FILE && $_FILES['movie-poster']['error'] !== UPLOAD_ERR_OK) {
             $errors[] = 'Poster upload error: ' . $_FILES['movie-poster']['error']; // More detailed error
         }
    }

    // 4. Handle File Uploads (Trailer File - Optional)
    $trailer_file_path = null;
    // Check if a trailer file was uploaded and if there's no error
    if (isset($_FILES['trailer-file']) && $_FILES['trailer-file']['error'] === UPLOAD_ERR_OK) {
        $trailerFile = $_FILES['trailer-file'];
        $allowedTypes = ['video/mp4', 'video/webm', 'video/ogg', 'video/quicktime']; // Allowed video types
        $maxFileSize = 50 * 1024 * 1024; // 50MB

        if (!in_array($trailerFile['type'], $allowedTypes)) {
            $errors[] = 'Invalid trailer file type. Only MP4, WebM, Ogg, MOV are allowed.';
        }
        if ($trailerFile['size'] > $maxFileSize) {
            $errors[] = 'Trailer file is too large. Maximum size is 50MB.';
        }

        // If no file errors yet, process the upload
        if (empty($errors)) {
            $fileExtension = pathinfo($trailerFile['name'], PATHINFO_EXTENSION);
            $newFileName = uniqid('trailer_', true) . '.' . $fileExtension;
            $destination = UPLOAD_DIR_TRAILERS . $newFileName;

            if (move_uploaded_file($trailerFile['tmp_name'], $destination)) {
                $trailer_file_path = $newFileName; // Store just the filename/relative path
                $trailer_url = null; // If file is uploaded, clear the URL
            } else {
                $errors[] = 'Failed to upload trailer file.';
            }
        }
    } else {
         // Handle specific upload errors for trailer file
         if (isset($_FILES['trailer-file']) && $_FILES['trailer-file']['error'] !== UPLOAD_ERR_NO_FILE && $_FILES['trailer-file']['error'] !== UPLOAD_ERR_OK) {
             $errors[] = 'Trailer file upload error: ' . $_FILES['trailer-file']['error']; // More detailed error
         }
    }

    // If both trailer URL and trailer file are empty
    if (empty($trailer_url) && empty($trailer_file_path)) {
        $errors[] = 'Either a Trailer URL or a Trailer File is required.';
    }


    // 5. If no errors, insert into database
    if (empty($errors)) {
        try {
            $pdo->beginTransaction();

            // Insert movie into movies table
            $movieId = createMovie(
                $title,
                $summary,
                $release_date,
                $duration_hours,
                $duration_minutes,
                $age_rating,
                $poster_image_path, // Use the uploaded file path
                $trailer_url,       // Use the URL if file not uploaded
                $trailer_file_path, // Use the file path if uploaded
                $userId             // Log the uploader
            );

            // Insert genres into movie_genres table
            foreach ($genres as $genre) {
                addMovieGenre($movieId, $genre);
            }

            $pdo->commit();

            $_SESSION['success_message'] = 'Movie uploaded successfully!';
            header('Location: indeks.php'); // Redirect to manage page
            exit;

        } catch (PDOException $e) {
            $pdo->rollBack();
            error_log("Database error during movie upload: " . $e->getMessage());
            $error_message = 'An internal error occurred while saving the movie.';

            // Clean up uploaded files if DB insertion failed
            if ($poster_image_path && file_exists(UPLOAD_DIR_POSTERS . $poster_image_path)) {
                unlink(UPLOAD_DIR_POSTERS . $poster_image_path);
            }
             if ($trailer_file_path && file_exists(UPLOAD_DIR_TRAILERS . $trailer_file_path)) {
                unlink(UPLOAD_DIR_TRAILERS . $trailer_file_path);
            }
        }
    } else {
        // If there were validation or file upload errors
        $error_message = implode('<br>', $errors);
    }
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Upload Movie - RatingTales</title>
    <link rel="stylesheet" href="styles.css">
    <link rel="stylesheet" href="upload.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Using FA 6 -->
</head>
<body>
    <div class="container">
        <!-- Sidebar Navigation -->
        <nav class="sidebar">
            <h2 class="logo">RATE-TALES</h2>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li>
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favorites</span></a></li>
                <li><a href="../review/index.php"><i class="fas fa-star"></i> <span>Review</span></a></li>
                <li><a href="indeks.php" class="active"><i class="fas fa-film"></i> <span>Manage</span></a></li> <!-- Corrected link to indeks.php -->
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
            </ul>
            <ul class="bottom-links">
                <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Corrected logout link -->
            </ul>
        </nav>

        <!-- Main Content -->
        <main class="main-content">
            <div class="upload-container">
                <h1>Upload New Movie</h1>

                 <?php if ($success_message): ?>
                    <div class="alert success"><?php echo htmlspecialchars($success_message); ?></div>
                <?php endif; ?>
                <?php if ($error_message): ?>
                    <div class="alert error"><?php echo $error_message; // HTML allowed for <br> ?></div>
                <?php endif; ?>

                <form class="upload-form" action="upload.php" method="post" enctype="multipart/form-data">
                    <div class="form-layout">
                        <div class="form-main">
                            <div class="form-group">
                                <label for="movie-title">Movie Title</label>
                                <input type="text" id="movie-title" name="movie-title" required value="<?php echo htmlspecialchars($title ?? ''); ?>">
                            </div>

                            <div class="form-group">
                                <label for="movie-summary">Movie Summary</label>
                                <textarea id="movie-summary" name="movie-summary" rows="4" required><?php echo htmlspecialchars($summary ?? ''); ?></textarea>
                            </div>

                            <div class="form-group">
                                <label>Genre</label>
                                <div class="genre-options">
                                    <?php
                                    // List all possible genres from DB ENUM or a predefined list
                                    $all_genres = ['action', 'adventure', 'comedy', 'drama', 'horror', 'supernatural', 'animation', 'sci-fi']; // Must match DB ENUM
                                    foreach ($all_genres as $genre_option):
                                        $checked = in_array($genre_option, $genres ?? []) ? 'checked' : '';
                                    ?>
                                    <label class="checkbox-label">
                                        <input type="checkbox" name="genre[]" value="<?php echo $genre_option; ?>" <?php echo $checked; ?>>
                                        <span><?php echo ucwords($genre_option); ?></span>
                                    </label>
                                    <?php endforeach; ?>
                                </div>
                            </div>

                            <div class="form-group">
                                <label for="age-rating">Age Rating</label>
                                <select id="age-rating" name="age-rating" required>
                                    <option value="">Select age rating</option>
                                    <?php
                                    // List allowed age ratings from DB ENUM or a predefined list
                                    $age_ratings = ['G', 'PG', 'PG-13', 'R', 'NC-17']; // Must match DB ENUM
                                    foreach ($age_ratings as $rating_option):
                                        $selected = (($age_rating ?? '') === $rating_option) ? 'selected' : '';
                                    ?>
                                        <option value="<?php echo $rating_option; ?>" <?php echo $selected; ?>><?php echo $rating_option; ?></option>
                                    <?php endforeach; ?>
                                </select>
                            </div>

                            <div class="form-group">
                                <label for="movie-trailer">Movie Trailer</label>
                                <div class="trailer-input">
                                    <input type="text" id="trailer-link" name="trailer-link" placeholder="Enter YouTube video URL" value="<?php echo htmlspecialchars($trailer_url ?? ''); ?>">
                                    <span class="trailer-note">* Paste YouTube video URL</span>
                                </div>
                                <div class="trailer-upload">
                                    <input type="file" id="trailer-file" name="trailer-file" accept="video/*">
                                    <span class="trailer-note">* Or upload video file (Max 50MB)</span>
                                </div>
                                 <p class="trailer-note" style="margin-top: 10px;">Only one trailer source (URL or File) is needed.</p>
                            </div>
                        </div>

                        <div class="form-side">
                            <div class="poster-upload">
                                <label for="movie-poster">Movie Poster</label>
                                <div class="upload-area" id="upload-area">
                                    <i class="fas fa-image"></i>
                                    <p>Click or drag image here</p>
                                     <input type="file" id="movie-poster" name="movie-poster" accept="image/*" required>
                                    <img id="poster-preview" src="#" alt="Poster Preview" style="display: none; max-width: 100%; max-height: 100%; object-fit: contain;"> <!-- Image preview -->
                                </div>
                                <p class="trailer-note" style="margin-top: 5px;">(Recommended: Aspect Ratio 2:3, Max 5MB)</p>
                            </div>

                            <div class="advanced-settings">
                                <h3>Advanced Settings</h3>
                                <div class="form-group">
                                    <label for="release-date">Release Date</label>
                                    <input type="date" id="release-date" name="release-date" required value="<?php echo htmlspecialchars($release_date ?? ''); ?>">
                                </div>

                                <div class="form-group">
                                    <label for="duration-hours">Film Duration</label>
                                    <div class="duration-inputs">
                                        <div class="duration-field">
                                            <input type="number" id="duration-hours" name="duration-hours" min="0" placeholder="Hours" required value="<?php echo htmlspecialchars($duration_hours ?? ''); ?>">
                                            <span>Hours</span>
                                        </div>
                                        <div class="duration-field">
                                            <input type="number" id="duration-minutes" name="duration-minutes" min="0" max="59" placeholder="Minutes" required value="<?php echo htmlspecialchars($duration_minutes ?? ''); ?>">
                                            <span>Minutes</span>
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>

                    <div class="form-actions">
                        <button type="button" class="cancel-btn" onclick="window.location.href='indeks.php'">Cancel</button> <!-- Corrected link -->
                        <button type="submit" class="submit-btn">Upload Movie</button>
                    </div>
                </form>
            </div>
        </main>
    </div>

    <script>
        // JavaScript for poster preview
        const posterInput = document.getElementById('movie-poster');
        const posterPreview = document.getElementById('poster-preview');
        const uploadArea = document.getElementById('upload-area');
        const uploadAreaIcon = uploadArea.querySelector('i');
        const uploadAreaText = uploadArea.querySelector('p');

        posterInput.addEventListener('change', function() {
            const file = this.files[0];
            if (file) {
                const reader = new FileReader();
                reader.onload = function(e) {
                    posterPreview.src = e.target.result;
                    posterPreview.style.display = 'block';
                    uploadAreaIcon.style.display = 'none'; // Hide icon
                    uploadAreaText.style.display = 'none'; // Hide text
                }
                reader.readAsDataURL(file); // Read the file as a data URL
            } else {
                // Reset if no file is selected
                posterPreview.src = '#';
                posterPreview.style.display = 'none';
                 uploadAreaIcon.style.display = ''; // Show icon
                 uploadAreaText.style.display = ''; // Show text
            }
        });

         // Optional: Handle drag and drop
         uploadArea.addEventListener('dragover', (e) => {
             e.preventDefault();
             uploadArea.style.borderColor = '#00ffff'; // Highlight drag area
         });

         uploadArea.addEventListener('dragleave', (e) => {
             e.preventDefault();
             uploadArea.style.borderColor = '#363636'; // Revert border color
         });

         uploadArea.addEventListener('drop', (e) => {
             e.preventDefault();
             uploadArea.style.borderColor = '#363636'; // Revert border color
             const files = e.dataTransfer.files;
             if (files.length > 0) {
                 posterInput.files = files; // Assign files to input
                 posterInput.dispatchEvent(new Event('change')); // Trigger change event
             }
         });


         // JavaScript to handle mutual exclusivity of trailer URL and File
         const trailerLinkInput = document.getElementById('trailer-link');
         const trailerFileInput = document.getElementById('trailer-file');

         trailerLinkInput.addEventListener('input', function() {
             if (this.value.trim() !== '') {
                 trailerFileInput.disabled = true; // Disable file input if URL is entered
             } else {
                 trailerFileInput.disabled = false; // Enable file input if URL is empty
             }
         });

         trailerFileInput.addEventListener('change', function() {
             if (this.files.length > 0) {
                 trailerLinkInput.disabled = true; // Disable URL input if file is selected
             } else {
                 trailerLinkInput.disabled = false; // Enable URL input if file selection is cleared
             }
         });

         // Initial check on page load (useful if values are pre-filled during edit)
         document.addEventListener('DOMContentLoaded', () => {
             if (trailerLinkInput.value.trim() !== '') {
                 trailerFileInput.disabled = true;
             } else if (trailerFileInput.files.length > 0) {
                 trailerLinkInput.disabled = true;
             }

             // If editing and poster already exists, show it
             // This requires fetching movie data in PHP first
             // Example (assuming you passed an existing poster URL via PHP):
             // <?php if (!empty($poster_image_url)): ?>
             //     posterPreview.src = '<?php echo htmlspecialchars($poster_image_url); ?>';
             //     posterPreview.style.display = 'block';
             //     uploadAreaIcon.style.display = 'none';
             //     uploadAreaText.style.display = 'none';
             // <?php endif; ?>
         });
    </script>
</body>
</html>
```

**18. `manage/upload.css` (Modified)**

Apply consistent styling.

```css
/* manage/upload.css */
/* Assuming base styles from manage/styles.css are included */

.upload-container {
    padding: 20px;
}

.upload-container h1 {
    color: #00ffff;
    font-size: 28px; /* Slightly larger */
    margin-bottom: 25px;
}

.upload-form {
    background-color: #242424;
    border-radius: 15px;
    padding: 30px;
}

.form-layout {
    display: grid;
    grid-template-columns: 2fr 1fr;
    gap: 30px;
     /* Responsive adjustments */
     @media (max-width: 992px) { /* Adjust breakpoint */
         grid-template-columns: 1fr; /* Stack columns on smaller screens */
     }
}

/* Form Main Section */
.form-main {
    display: flex;
    flex-direction: column;
    gap: 20px; /* Reduced gap */
}

/* Form Side Section */
.form-side {
    display: flex;
    flex-direction: column;
    gap: 20px; /* Reduced gap */
}

/* Form Groups */
.form-group {
    margin-bottom: 0; /* Use gap in flex/grid instead */
}

.form-group label {
    display: block;
    margin-bottom: 8px;
    color: #ffffff;
    font-size: 15px; /* Slightly smaller font */
    font-weight: bold; /* Make labels bold */
}

.duration-inputs {
    display: flex;
    gap: 15px;
}

.duration-field {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 5px;
}

.duration-field span {
    color: #888; /* Gray color */
    font-size: 12px;
}

.form-group input[type="text"],
.form-group input[type="number"],
.form-group input[type="date"],
.form-group select,
.form-group textarea {
    width: 100%;
    padding: 12px;
    background-color: #1a1a1a;
    border: 1px solid #363636;
    border-radius: 8px;
    color: #ffffff;
    font-size: 14px;
    transition: border-color 0.3s, box-shadow 0.3s;
     font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; /* Match body font */
}

.form-group input:focus,
.form-group select:focus,
.form-group textarea:focus {
    border-color: #00ffff;
    box-shadow: 0 0 0 2px rgba(0, 255, 255, 0.2);
    outline: none;
}

.form-group textarea {
    min-height: 120px;
    resize: vertical;
}

/* Poster Upload */
.poster-upload {
    position: relative;
    /* Removed height restriction to allow dynamic height based on upload-area */
}

.upload-area {
    width: 100%;
    height: 300px; /* Fixed height for upload area */
    background-color: #1a1a1a;
    border: 2px dashed #363636;
    border-radius: 8px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    transition: border-color 0.3s, background-color 0.3s;
    margin-top: 10px;
     position: relative; /* Needed for positioning internal elements */
    overflow: hidden; /* Hide overflowing preview */
}

.upload-area:hover {
    border-color: #00ffff;
    background-color: #2a2a2a;
}

.upload-area i {
    font-size: 48px;
    color: #666;
    margin-bottom: 10px;
     transition: color 0.3s;
}
.upload-area:hover i {
    color: #00ffff;
}


.upload-area p {
    color: #666;
    font-size: 14px;
     transition: color 0.3s;
}
.upload-area:hover p {
    color: #ffffff;
}


.upload-area input[type="file"] {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    opacity: 0;
    cursor: pointer;
     z-index: 10; /* Ensure input is clickable */
}

/* Poster preview inside upload area */
#poster-preview {
    display: none; /* Hidden by default */
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover; /* Cover the area */
     z-index: 5; /* Below the input, above icon/text */
}


/* Advanced Settings */
.form-section {
    margin-top: 30px; /* Space above section */
}

.form-section h2, .advanced-settings h3 { /* Style for sub-headers */
    color: #00ffff;
    font-size: 20px;
    margin-bottom: 20px;
     border-bottom: 1px solid #333; /* Separator */
     padding-bottom: 10px;
}

.advanced-settings {
    display: flex; /* Use flexbox for simplicity */
    flex-direction: column;
    gap: 20px;
    background-color: #1a1a1a; /* Background for advanced settings block */
    border-radius: 10px;
    padding: 20px;
}


/* Genre Options */
.genre-options {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(100px, 1fr)); /* Adjust min width */
    gap: 10px;
}

.checkbox-label {
    display: flex;
    align-items: center;
    cursor: pointer;
    user-select: none; /* Prevent text selection */
}

.checkbox-label input[type="checkbox"] {
    margin-right: 8px;
    appearance: none; /* Hide default checkbox */
    width: 18px;
    height: 18px;
    border: 2px solid #363636;
    border-radius: 4px;
    background-color: #1a1a1a;
    cursor: pointer;
    transition: all 0.3s;
    flex-shrink: 0; /* Prevent checkbox from shrinking */
     position: relative; /* For custom checkmark */
}

.checkbox-label input[type="checkbox"]:checked {
    background-color: #00ffff;
    border-color: #00ffff;
}

.checkbox-label input[type="checkbox"]:checked::before {
    content: '\f00c'; /* Font Awesome check icon */
    font-family: 'Font Awesome 6 Free'; /* Specify font family */
    font-weight: 900; /* Use solid icon */
    display: flex;
    justify-content: center;
    align-items: center;
    color: #1a1a1a; /* Dark color for checkmark */
    font-size: 12px;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}

.checkbox-label input[type="checkbox"]:disabled {
    opacity: 0.6;
    cursor: not-allowed;
}


.checkbox-label span {
    color: #ffffff;
    font-size: 14px;
}

/* Trailer Input */
.trailer-input {
    position: relative;
    margin-bottom: 10px;
}

.trailer-input input[type="text"] {
    padding-right: 120px; /* Make space for the note */
}

.trailer-upload {
    position: relative;
    background-color: #1a1a1a;
    border: 1px solid #363636;
    border-radius: 8px;
    padding: 12px;
    margin-top: 10px;
    display: flex; /* Use flex for layout */
    align-items: center;
    gap: 10px; /* Space between file input and note */
}

.trailer-upload input[type="file"] {
    width: auto; /* Auto width */
    flex-grow: 1; /* Allow file input to grow */
    color: #ffffff;
    font-size: 14px;
    padding: 0; /* Remove default input padding */
    border: none; /* Remove default input border */
    background: none; /* Remove default input background */
}

.trailer-upload input[type="file"]::file-selector-button {
     /* Style the file selector button */
    background-color: #363636;
    color: #ffffff;
    padding: 8px 12px;
    border: none;
    border-radius: 5px;
    margin-right: 15px;
    cursor: pointer;
    transition: background-color 0.3s;
}
.trailer-upload input[type="file"]::file-selector-button:hover {
    background-color: #444444;
}

.trailer-note {
    color: #888;
    font-size: 12px;
    white-space: nowrap; /* Prevent note from wrapping */
    flex-shrink: 0; /* Prevent note from shrinking */
}
.trailer-input .trailer-note { /* Specific note positioning for URL input */
    position: absolute;
    right: 12px;
    top: 50%;
    transform: translateY(-50%);
}
.trailer-upload .trailer-note { /* Specific note positioning for file input */
    position: static; /* Remove absolute positioning */
    display: block;
    margin-top: 5px;
    transform: none;
    text-align: right; /* Align note to the right */
    width: 100%; /* Take full width */
}


/* Form Actions */
.form-actions {
    display: flex;
    justify-content: flex-end;
    gap: 15px;
    margin-top: 30px;
}

.form-actions button {
    padding: 12px 24px;
    border: none;
    border-radius: 8px;
    font-size: 14px;
    font-weight: bold; /* Make buttons bold */
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 8px;
    transition: all 0.3s ease;
}

.form-actions button:hover {
    transform: translateY(-2px);
}

.cancel-btn {
    background-color: #363636; /* Dark gray cancel */
    color: #ffffff;
}

.cancel-btn:hover {
    background-color: #444444;
}

.submit-btn {
    background-color: #00ffff; /* Teal submit */
    color: #1a1a1a;
}

.submit-btn:hover {
    background-color: #00cccc;
}


/* Alert Messages (Copy from favorite/styles.css) */
.alert {
    padding: 10px 20px;
    margin-bottom: 20px;
    border-radius: 8px;
    font-size: 14px;
    text-align: center;
}

.alert.success {
    background-color: #00ff0033;
    color: #00ff00;
    border: 1px solid #00ff0088;
}

.alert.error {
    background-color: #ff000033;
    color: #ff0000;
    border: 1px solid #ff000088;
}

/* Scrollbar Styles (Copy from beranda/styles.css) */
.main-content::-webkit-scrollbar {
    width: 8px;
}

.main-content::-webkit-scrollbar-track {
    background: #242424;
}

.main-content::-webkit-scrollbar-thumb {
    background: #363636;
    border-radius: 4px;
}

.main-content::-webkit-scrollbar-thumb:hover {
    background: #00ffff;
}


/* Responsive Design */
@media (max-width: 992px) { /* Adjust breakpoint */
    .form-layout {
        grid-template-columns: 1fr; /* Stack columns */
        gap: 25px; /* Gap between stacked sections */
    }

    .form-side {
         gap: 20px;
    }

    .form-actions {
        flex-direction: column;
        gap: 10px; /* Reduced gap */
    }

    .form-actions button {
        width: 100%;
        justify-content: center;
    }

     .poster-upload {
         margin-top: 0; /* Remove top margin when stacked */
     }
     .advanced-settings {
         margin-top: 0; /* Remove top margin when stacked */
         padding-top: 20px; /* Add padding top */
         border-top: 1px solid #333; /* Add separator */
     }
     .advanced-settings h3 {
         margin-top: 0;
     }

}

```

**19. `acc_page/index.php` (Modified)**

Fetch user profile details and uploaded movies from the database. Implement profile update logic (via AJAX or form). Update navigation.

```php
<?php
// acc_page/index.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated
redirectIfNotAuthenticated();

// Get authenticated user ID
$userId = $_SESSION['user_id'];

// Fetch authenticated user details from the database
$user = getAuthenticatedUser();

// Fetch movies uploaded by the current user
$uploadedMovies = getUserUploadedMovies($userId);

// Handle profile update (using AJAX or a form POST)
// For simplicity, let's implement a POST handler for bio and potentially other fields.
// A dedicated AJAX handler would be better for a smoother UX for inline editing.

if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['update_profile'])) {
    $update_data = [];
    $has_update = false;

    // Handle bio update
    if (isset($_POST['bio'])) {
        $update_data['bio'] = trim($_POST['bio']);
        $has_update = true;
    }

     // Handle display name update (if implemented in the form)
     // if (isset($_POST['display_name'])) {
     //     $update_data['full_name'] = trim($_POST['display_name']);
     //     $has_update = true;
     // }

     // Handle profile image upload (more complex, needs file handling)
     // if (isset($_FILES['profile_image']) && $_FILES['profile_image']['error'] === UPLOAD_ERR_OK) {
     //     // Process file upload, save path, add to $update_data['profile_image']
     //     // ... file upload logic similar to movie poster ...
     //     $has_update = true;
     // }


    if ($has_update) {
        if (updateUser($userId, $update_data)) {
            $_SESSION['success_message'] = 'Profile updated successfully!';
             // Refresh user data after update
            $user = getAuthenticatedUser();
        } else {
            $_SESSION['error_message'] = 'Failed to update profile.';
        }
    } else {
         $_SESSION['error_message'] = 'No data to update.';
    }

     // Redirect to prevent form resubmission and show messages
    header('Location: index.php');
    exit;
}


// Get messages from session
$success_message = isset($_SESSION['success_message']) ? $_SESSION['success_message'] : null;
unset($_SESSION['success_message']);
$error_message = isset($_SESSION['error_message']) ? $_SESSION['error_message'] : null;
unset($_SESSION['error_message']);

// Handle user not found after authentication check (shouldn't happen if getAuthenticatedUser works)
if (!$user) {
    // Fallback or severe error
    $_SESSION['error_message'] = 'User profile could not be loaded.';
    header('Location: ../autentikasi/logout.php'); // Force logout
    exit;
}

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Profile</title>
    <link rel="stylesheet" href="styles.css">
     <link rel="stylesheet" href="../review/styles.css"> <!-- Include review styles for movie card look -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Using FA 6 -->
</head>
<body>
    <div class="container">
        <nav class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li>
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favourites</span></a></li>
                <li><a href="../review/index.php"><i class="fas fa-star"></i> <span>Review</span></a></li>
                <li><a href="../manage/indeks.php"><i class="fas fa-film"></i> <span>Manage</span></a></li>
                 <li class="active"><a href="#"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Active Profile link -->
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Corrected logout link -->
                </ul>
            </div>
        </nav>
        <main class="main-content">
             <?php if ($success_message): ?>
                <div class="alert success"><?php echo htmlspecialchars($success_message); ?></div>
            <?php endif; ?>
            <?php if ($error_message): ?>
                <div class="alert error"><?php echo $error_message; // Allow HTML if <br> is used ?></div>
            <?php endif; ?>

            <div class="profile-header">
                <div class="profile-info">
                    <div class="profile-image">
                        <!-- Display user's actual profile image or fallback -->
                        <img src="<?php echo htmlspecialchars($user['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($user['full_name'] ?? $user['username']) . '&background=random'); ?>" alt="Profile Picture">
                        <!-- Optional: Add edit icon for profile image upload -->
                         <!-- <i class="fas fa-camera edit-icon" title="Change Profile Image"></i> -->
                    </div>
                    <div class="profile-details">
                        <h1>
                            <!-- Display full name, make it editable -->
                            <span id="displayName"><?php echo htmlspecialchars($user['full_name'] ?? 'N/A'); ?></span>
                            <!-- Edit icon triggers JS/form to edit full name -->
                            <!-- <i class="fas fa-pen edit-icon" onclick="toggleEdit('displayName')"></i> -->
                        </h1>
                        <p class="username">
                            <!-- Display username (usually not editable, but kept original structure) -->
                            @<span id="username"><?php echo htmlspecialchars($user['username']); ?></span>
                            <!-- Edit icon for username (usually disabled or hidden) -->
                            <!-- <i class="fas fa-pen edit-icon" onclick="toggleEdit('username')"></i> -->
                        </p>
                         <p class="user-meta"><?php echo htmlspecialchars($user['age'] ?? 'N/A'); ?> | <?php echo htmlspecialchars($user['gender'] ?? 'N/A'); ?></p> <!-- Display age and gender -->
                    </div>
                </div>
                <div class="about-me">
                    <h2>ABOUT ME:</h2>
                    <!-- Bio section - make it editable -->
                    <div class="about-content" id="bio" onclick="enableBioEdit()">
                         <?php echo nl2br(htmlspecialchars($user['bio'] ?? 'Click to add bio...')); ?>
                    </div>
                </div>
            </div>
            <div class="posts-section">
                <h2>Movies I Uploaded</h2> <!-- Changed title -->
                <div class="movies-grid review-grid"> <!-- Use review-grid class -->
                    <?php if (!empty($uploadedMovies)): ?>
                        <?php foreach ($uploadedMovies as $movie): ?>
                            <div class="movie-card" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'"> <!-- Link to movie details -->
                                <div class="movie-poster">
                                    <img src="<?php echo htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                    <div class="movie-actions">
                                         <!-- Optional: Add edit/delete action buttons directly here -->
                                         <!-- <a href="../manage/edit.php?id=<?php echo $movie['movie_id']; ?>" class="action-btn" title="Edit Movie"><i class="fas fa-edit"></i></a> -->
                                         <!-- Delete form -->
                                         <!-- <form action="../manage/indeks.php" method="POST" onsubmit="return confirm('Delete movie?');">
                                             <input type="hidden" name="delete_movie_id" value="<?php echo $movie['movie_id']; ?>">
                                             <button type="submit" class="action-btn" title="Delete Movie"><i class="fas fa-trash"></i></button>
                                         </form> -->
                                    </div>
                                </div>
                                <div class="movie-details">
                                    <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                                     <p class="movie-info"><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?></p>
                                    <div class="rating">
                                        <div class="stars">
                                             <?php
                                            // Display average rating stars
                                            $average_rating = floatval($movie['average_rating']);
                                            for ($i = 1; $i <= 5; $i++) {
                                                if ($i <= $average_rating) {
                                                    echo '<i class="fas fa-star"></i>';
                                                } else if ($i - 0.5 <= $average_rating) {
                                                    echo '<i class="fas fa-star-half-alt"></i>';
                                                } else {
                                                    echo '<i class="far fa-star"></i>';
                                                }
                                            }
                                            ?>
                                        </div>
                                        <span class="rating-count">(<?php echo htmlspecialchars($movie['average_rating']); ?>)</span>
                                    </div>
                                </div>
                            </div>
                        <?php endforeach; ?>
                    <?php else: ?>
                        <div class="empty-state full-width"> <!-- Use full-width empty state -->
                            <i class="fas fa-film"></i>
                            <p>You haven't uploaded any movies yet.</p>
                            <p class="subtitle">Go to the <a href="../manage/indeks.php">Manage</a> section to upload your first movie.</p>
                        </div>
                    <?php endif; ?>
                </div>
            </div>
        </main>
    </div>
    <script>
        // JavaScript for bio editing
        function enableBioEdit() {
            const bioElement = document.getElementById('bio');
            const currentText = bioElement.textContent.trim();

            // Check if already in edit mode
            if (bioElement.querySelector('textarea')) {
                return;
            }

            const textarea = document.createElement('textarea');
            textarea.className = 'edit-input bio-input';
            textarea.value = (currentText === 'Click to add bio...' || currentText === '') ? '' : currentText;
            textarea.placeholder = 'Write something about yourself...';

            // Create Save and Cancel buttons
            const saveButton = document.createElement('button');
            saveButton.textContent = 'Save';
            saveButton.className = 'btn-save-bio'; // Add a class for styling

            const cancelButton = document.createElement('button');
            cancelButton.textContent = 'Cancel';
            cancelButton.className = 'btn-cancel-bio'; // Add a class for styling
            cancelButton.type = 'button'; // Prevent form submission

            // Create a form to wrap the textarea and buttons
            const form = document.createElement('form');
            form.method = 'POST';
            form.action = 'index.php'; // Submit back to this page

            const hiddenInput = document.createElement('input');
            hiddenInput.type = 'hidden';
            hiddenInput.name = 'update_profile'; // Identifier for the POST handler
            hiddenInput.value = '1';

            const bioInputName = document.createElement('input');
            bioInputName.type = 'hidden';
            bioInputName.name = 'bio'; // Name for the bio data


            // Replace content with textarea and buttons
            bioElement.textContent = '';
            form.appendChild(hiddenInput);
            form.appendChild(textarea);
             // Wrap buttons in a div for layout
             const buttonDiv = document.createElement('div');
             buttonDiv.style.marginTop = '10px';
             buttonDiv.style.textAlign = 'right'; // Align buttons to the right
             buttonDiv.appendChild(cancelButton);
             buttonDiv.appendChild(saveButton);
             form.appendChild(buttonDiv);

            bioElement.appendChild(form);
            textarea.focus();

            // Handle Save
            saveButton.onclick = function() {
                 bioInputName.value = textarea.value.trim(); // Set the value of the hidden input
                 form.appendChild(bioInputName); // Append the bio input to the form
                form.submit(); // Submit the form
            };

            // Handle Cancel
            cancelButton.onclick = function() {
                let originalValue = (currentText === 'Click to add bio...' || currentText === '') ? 'Click to add bio...' : currentText;
                bioElement.innerHTML = nl2br(htmlspecialchars(originalValue)); // Restore original content
            };

             // Handle blur (optional, can also save on blur)
             // textarea.onblur = function() {
             //    // Might auto-save here or just revert/keep changes based on preference
             //    // For now, require clicking Save/Cancel
             // };
        }

         // Helper function for nl2br (client-side equivalent for display)
         function nl2br(str) {
             if (typeof str !== 'string') return str;
             return str.replace(/(?:\r\n|\r|\n)/g, '<br>');
         }
          // Helper function for HTML escaping (client-side)
         function htmlspecialchars(str) {
             if (typeof str !== 'string') return str;
             return str.replace(/&/g, '&amp;')
                       .replace(/</g, '&lt;')
                       .replace(/>/g, '&gt;')
                       .replace(/"/g, '&quot;')
                       .replace(/'/g, '&#039;');
         }
    </script>
     <style>
         /* Add style for bio edit state buttons */
         .btn-save-bio, .btn-cancel-bio {
             padding: 8px 15px;
             border: none;
             border-radius: 5px;
             cursor: pointer;
             font-size: 14px;
             margin-left: 10px;
             transition: background-color 0.3s;
         }
         .btn-save-bio {
             background-color: #00ffff;
             color: #1a1a1a;
         }
         .btn-save-bio:hover {
             background-color: #00cccc;
         }
         .btn-cancel-bio {
             background-color: #555;
             color: #fff;
         }
         .btn-cancel-bio:hover {
             background-color: #666;
         }

         /* Style for user meta info */
         .user-meta {
             color: #888;
             font-size: 14px;
             margin-top: 5px;
         }
         /* Style for movies grid on profile page */
         .movies-grid.review-grid {
            padding: 0; /* Remove padding added by review-grid if needed */
         }

     </style>
</body>
</html>
```

**20. `acc_page/styles.css` (Modified)**

Apply consistent styling, especially the sidebar and header profile info.

```css
/* acc_page/styles.css */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

body {
    background-color: #1a1a1a;
    color: #ffffff;
}

.container {
    display: flex;
    min-height: 100vh;
}

/* Sidebar Styles (Copy from beranda/styles.css for consistency) */
.sidebar {
    width: 250px;
    background-color: #242424;
    padding: 20px;
    display: flex;
    flex-direction: column;
    position: fixed;
    height: 100vh;
    top: 0;
    left: 0;
    z-index: 100;
    box-shadow: 2px 0 5px rgba(0,0,0,0.3);
}

.logo h2 {
    color: #00ffff;
    margin-bottom: 40px;
    font-size: 24px;
     text-align: center;
}

.nav-links {
    list-style: none;
    flex-grow: 1;
     padding-top: 20px;
}

.nav-links li, .bottom-links li {
    margin-bottom: 15px;
}

.nav-links a, .bottom-links a {
    color: #ffffff;
    text-decoration: none;
    display: flex;
    align-items: center;
    padding: 12px 15px;
    border-radius: 8px;
    transition: background-color 0.3s, color 0.3s;
    font-size: 15px;
}

.nav-links a:hover, .bottom-links a:hover {
    background-color: #363636;
    color: #00ffff;
}

.nav-links i, .bottom-links i {
    margin-right: 15px;
    width: 20px;
     text-align: center;
}

.active a {
    background-color: #363636;
    color: #00ffff;
    font-weight: bold;
}
.active a i {
    color: #00ffff;
}


.bottom-links {
    margin-top: auto;
    list-style: none;
    padding-top: 20px;
    border-top: 1px solid #363636;
}


/* Main Content Styles */
.main-content {
    flex-grow: 1;
    margin-left: 250px;
    padding: 20px;
    max-height: 100vh;
    overflow-y: auto;
}

/* Profile Header Styles */
.profile-header {
    background-color: #242424;
    border-radius: 15px;
    padding: 30px;
    margin-bottom: 30px;
}

.profile-info {
    display: flex;
    align-items: center;
    gap: 30px;
    margin-bottom: 30px;
    flex-wrap: wrap; /* Allow wrapping on smaller screens */
}

.profile-image {
    width: 120px;
    height: 120px;
    border-radius: 50%;
    overflow: hidden;
    border: 3px solid #00ffff; /* Teal border */
    box-shadow: 0 0 10px rgba(0, 255, 255, 0.5); /* Teal glow */
    flex-shrink: 0; /* Prevent shrinking */
}

.profile-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.profile-details {
     flex-grow: 1; /* Allow details to fill space */
     min-width: 200px; /* Ensure details don't collapse too much */
}

.profile-details h1 {
    font-size: 32px;
    margin-bottom: 5px; /* Reduced margin */
    display: flex;
    align-items: center;
    gap: 15px;
     color: #00ffff; /* Teal color for name */
}

.edit-icon {
    font-size: 18px; /* Slightly larger icon */
    color: #00ffff;
    cursor: pointer;
    transition: color 0.3s, transform 0.2s;
    margin-left: 5px; /* Reduced margin */
    opacity: 0.8;
}

.edit-icon:hover {
    color: #00cccc;
    opacity: 1;
    transform: scale(1.1);
}

/* Styles for the edit state (textarea/input) */
.edit-input {
    background-color: #363636;
    border: 1px solid #00ffff;
    border-radius: 4px;
    color: #ffffff;
    padding: 8px 12px;
    font-size: inherit;
    width: auto;
    outline: none;
    font-family: inherit;
    transition: border-color 0.3s, box-shadow 0.3s;
}

.edit-input:focus {
    border-color: #00cccc;
    box-shadow: 0 0 0 2px rgba(0, 255, 255, 0.2);
}

.bio-input {
    width: 100% !important;
    min-height: 100px;
    resize: vertical;
    line-height: 1.5;
}

.bio-input::placeholder {
    color: #666666;
    font-style: italic;
}

/* Style for the display bio area */
#bio {
    cursor: pointer;
    transition: background-color 0.3s;
     padding: 10px; /* Add padding to clickable area */
     border-radius: 8px;
     line-height: 1.6;
     color: #ccc; /* Lighter text color */
}

#bio:hover {
    background-color: #2a2a2a;
}

.username {
    color: #888;
    font-size: 16px;
    margin-bottom: 5px; /* Space below username */
}

.about-me {
    background-color: #1a1a1a;
    border-radius: 10px;
    padding: 20px;
}

.about-me h2 {
    color: #00ffff;
    margin-bottom: 15px;
    font-size: 20px;
     border-bottom: 1px solid #333; /* Separator */
     padding-bottom: 10px;
}

.about-content {
    min-height: 100px;
    background-color: #242424;
    border-radius: 8px;
    padding: 15px;
    white-space: pre-wrap; /* Preserve line breaks from nl2br */
}

/* Posts Section Styles */
.posts-section {
    padding: 20px 0; /* Remove left/right padding to align grid */
}

.posts-section h2 {
    color: #00ffff;
    margin-bottom: 20px;
    font-size: 24px;
}

/* Movies Grid Styles (Uses .review-grid from review/styles.css now) */
/* .posts-grid { ... removed ... } */

/* Post Card Styles (Uses .movie-card etc. from review/styles.css now) */
/* .post-card { ... removed ... } */

/* Empty State Styles (Copy from favorite/styles.css) */
.empty-state {
    grid-column: 1 / -1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 40px;
    text-align: center;
    background-color: #242424;
    border-radius: 15px;
    margin: 20px 0;
     color: #ffffff;
}

.empty-state i {
    font-size: 4em;
    color: #00ffff;
    margin-bottom: 20px;
}

.empty-state p {
    margin: 5px 0;
    font-size: 1.2em;
}

.empty-state .subtitle {
    color: #666;
    font-size: 0.9em;
}
.empty-state .subtitle a {
     color: #00ffff;
     text-decoration: none;
     font-weight: bold;
}
.empty-state .subtitle a:hover {
     text-decoration: underline;
}

.empty-state.full-width {
    width: 100%;
    margin: 0;
     min-height: 300px;
}


/* Scrollbar Styles (Copy from beranda/styles.css) */
.main-content::-webkit-scrollbar {
    width: 8px;
}

.main-content::-webkit-scrollbar-track {
    background: #242424;
}

.main-content::-webkit-scrollbar-thumb {
    background: #363636;
    border-radius: 4px;
}

.main-content::-webkit-scrollbar-thumb:hover {
    background: #00ffff;
}


```

**Instructions for Setup:**

1.  **Database:** Create the `ratingtales` database and run the updated SQL schema to create the tables (`users`, `movies`, `movie_genres`, `reviews`, `favorites`).
2.  **File Structure:** Create the directories `includes` and `uploads/posters`, `uploads/trailers`.
3.  **Place Files:**
    *   Place `config/database.php` in the `config` directory.
    *   Place `includes/config.php` in the `includes` directory.
    *   Place the modified PHP and CSS files in their respective directories (`autentikasi`, `beranda`, `favorite`, `manage`, `review`, `acc_page`).
    *   Place the original `autentikasi/animation.js` in the `autentikasi` directory.
    *   Create a placeholder image `gambar/placeholder.jpg` and the `gambar/5302920.jpg` background image, placing them in a new `gambar` directory at the root level (or adjust paths).
    *   Place the favicon `gambar/Untitled142_20250310223718.png` at the root level (or adjust path in login/register forms).
4.  **Server:** Ensure you have a web server (like Apache or Nginx) with PHP installed and configured to serve files from your project root. Make sure PHP has the PDO MySQL extension enabled and file uploads are permitted (`upload_max_filesize`, `post_max_size` in `php.ini`). The `uploads` directory needs write permissions for the web server user.
5.  **Access:** Navigate to `http://localhost/RatingTales/autentikasi/form-register.php` or `http://localhost/RatingTales/autentikasi/form-login.php` in your browser (adjust `/RatingTales/` if your project is in a different sub-directory).

**Key Changes and Security:**

*   **Database Integration:** All pages now fetch data from and submit data to the MySQL database using the PDO connection.
*   **SQL Injection Prevention:** **Crucially**, all database interactions use prepared statements (`$pdo->prepare()`, `$stmt->execute([])`) with placeholders (`?`). This is the primary defense against SQL injection. User input is never directly concatenated into SQL queries.
*   **Authentication:** `includes/config.php` provides `isAuthenticated()` and `redirectIfNotAuthenticated()`. Protected pages (`favorite`, `review`, `manage`, `acc_page`) now call `redirectIfNotAuthenticated()` at the beginning. User ID is stored in `$_SESSION['user_id']`.
*   **Password Hashing:** Registration uses `password_hash()` with BCRYPT. Login uses `password_verify()`.
*   **CAPTCHA:** CAPTCHA logic remains primarily server-side. The CAPTCHA code is stored and validated in the session. Client-side JS is only for drawing and refreshing the image. The actual validation happens in PHP when the form is submitted.
*   **File Uploads:** `manage/upload.php` now handles saving uploaded files (poster, trailer) to designated `uploads` directories and stores the *file paths* in the database. Basic file type and size validation are included. Unique filenames are generated to prevent overwriting and path traversal issues.
*   **Dynamic Data:** Hardcoded movie arrays are replaced by database queries. User details and uploaded movies are fetched from the DB.
*   **Icon Functionality:** Icons and buttons now trigger PHP actions (form submissions for delete/favorite toggle) or JS functions that lead to PHP actions (linking to details page, trailer modal). The profile edit icons have client-side JS hooks, but the save requires a form submission (implemented for bio).
*   **Navigation:** Sidebar links are updated to point to the correct PHP files. A 'Profile' link is added to the sidebar.
*   **Visual Consistency:** Sidebar styles and main content padding/margin are made consistent across pages. Empty state messages and styles are standardized.
*   **Review Details:** Clicking a movie card in the `review` list now links to `review/movie-details.php` with the movie ID in the URL (`?id=X`). The details page fetches the specific movie data, its average rating, comments, and trailer info from the DB. A simple form for submitting ratings and comments is added.
*   **Favorites:** Favorite status is checked against the DB. The favorite button on the details page toggles the favorite status via a GET request (can be changed to POST or AJAX). The favorites page lists only the current user's favorites.
*   **Manage:** Lists only the current user's uploaded movies. Delete functionality is added via a POST form.
*   **Profile:** Displays user's actual data (name, username, bio, age, gender, image) and their uploaded movies. Bio editing is implemented via a form POST.

This comprehensive update integrates the application with the database, enhances functionality, addresses security, and improves visual consistency according to your requirements. Remember to replace `"YOUR_GOOGLE_CLIENT_ID"` in `form-login.php` if you plan to implement Google Sign-In.
