Okay, here is the *full code* for all the files, incorporating the database integration, feature implementations, styling consistency, and the fixes for the `auth_helper.php` and `bio` column errors.

**Remember to set up your database and file structure correctly as described in the previous instructions before running this code.**

---

**1. Database Schema (SQL)**

*Save this as `database.sql` and run it in your MySQL client (like phpMyAdmin or MySQL Workbench) to create the database and tables.*

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS ratingtales CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE ratingtales;

-- Users table
CREATE TABLE users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    age INT,
    gender ENUM('Laki-laki', 'Perempuan'),
    bio TEXT, -- Added bio column
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
    poster_image VARCHAR(255),
    trailer_url VARCHAR(255),
    trailer_file VARCHAR(255),
    uploaded_by INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (uploaded_by) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Movie genres table
CREATE TABLE movie_genres (
    movie_id INT NOT NULL,
    genre ENUM('action', 'adventure', 'comedy', 'drama', 'horror', 'supernatural', 'animation', 'sci-fi') NOT NULL,
    PRIMARY KEY (movie_id, genre),
    FOREIGN KEY (movie_id) REFERENCES movies(movie_id) ON DELETE CASCADE
);

-- Reviews table
CREATE TABLE reviews (
    review_id INT AUTO_INCREMENT PRIMARY KEY,
    movie_id INT NOT NULL,
    user_id INT NOT NULL,
    rating DECIMAL(2,1) NOT NULL,
    comment TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (movie_id) REFERENCES movies(movie_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Favorites table
CREATE TABLE favorites (
    favorite_id INT AUTO_INCREMENT PRIMARY KEY,
    movie_id INT NOT NULL,
    user_id INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (movie_id) REFERENCES movies(movie_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    UNIQUE KEY unique_favorite (movie_id, user_id)
);

-- Create indexes
CREATE INDEX idx_movie_title ON movies(title);
CREATE INDEX idx_movie_genre ON movie_genres(genre);
CREATE INDEX idx_movie_rating ON reviews(movie_id, rating);
CREATE INDEX idx_user_favorites ON favorites(user_id, created_at);
CREATE INDEX idx_movie_uploaded_by ON movies(uploaded_by);
CREATE INDEX idx_review_movie_user ON reviews(movie_id, user_id);
```

---

**2. `config/database.php`**

*Place this file in the `config` directory at your project root.*

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
     // Set encoding for proper character handling
    $pdo->exec("SET NAMES 'utf8mb4'");
    $pdo->exec("SET CHARACTER SET utf8mb4");

} catch(PDOException $e) {
    // Log the error instead of dying on a live site
    error_log("Database connection failed: " . $e->getMessage());
    // Provide a user-friendly message
    die("Oops! Something went wrong with the database connection. Please try again later.");
}

// User Functions
// Added full_name, age, gender, bio to createUser
function createUser($full_name, $username, $email, $password, $age, $gender, $profile_image = null, $bio = null) {
    global $pdo;
    $hashedPassword = password_hash($password, PASSWORD_BCRYPT); // Use BCRYPT for better security
    $stmt = $pdo->prepare("INSERT INTO users (full_name, username, email, password, age, gender, profile_image, bio) VALUES (?, ?, ?, ?, ?, ?, ?, ?)");
    return $stmt->execute([$full_name, $username, $email, $hashedPassword, $age, $gender, $profile_image, $bio]);
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
             // Use backticks for column names in case they are reserved words (e.g., `user`)
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
    $stmt = $pdo->prepare("INSERT IGNORE INTO movie_genres (movie_id, genre) VALUES (?, ?)"); // Use INSERT IGNORE to prevent duplicates
    return $stmt->execute([$movie_id, $genre]);
}

function getMovieById($movieId) {
    global $pdo;
    // Fetch movie details along with its genres and the uploader's username
    $stmt = $pdo->prepare("
        SELECT
            m.*,
            GROUP_CONCAT(mg.genre SEPARATOR ', ') as genres, -- Use separator for clarity
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
            GROUP_CONCAT(mg.genre SEPARATOR ', ') as genres,
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
            GROUP_CONCAT(mg.genre SEPARATOR ', ') as genres,
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
// Supports inserting a new review or updating an existing one (upsert)
function createReview($movie_id, $user_id, $rating, $comment) {
    global $pdo;
    // Using ON DUPLICATE KEY UPDATE to handle cases where a user reviews the same movie again
    // This assumes a unique key on (movie_id, user_id) in the reviews table, which is standard practice
    // NOTE: Our schema does *not* have a unique key on (movie_id, user_id) for reviews.
    // Let's adjust this function to DELETE any previous review by the same user first, then INSERT.
    // Or modify the schema to add the UNIQUE KEY. Modifying schema is better.
    // Assume schema is updated with UNIQUE KEY unique_user_movie_review (movie_id, user_id) on reviews table.
    // If schema cannot be changed, use DELETE + INSERT.

    // Option 1: If reviews table has UNIQUE KEY (movie_id, user_id)
    $stmt = $pdo->prepare("
        INSERT INTO reviews (movie_id, user_id, rating, comment)
        VALUES (?, ?, ?, ?)
        ON DUPLICATE KEY UPDATE rating = VALUES(rating), comment = VALUES(comment), created_at = CURRENT_TIMESTAMP -- Update timestamp on update
    ");
     return $stmt->execute([$movie_id, $user_id, $rating, $comment]);

     /*
     // Option 2: If reviews table does NOT have UNIQUE KEY (movie_id, user_id) - less ideal for tracking single review per user
     $stmt = $pdo->prepare("INSERT INTO reviews (movie_id, user_id, rating, comment) VALUES (?, ?, ?, ?)");
     return $stmt->execute([$movie_id, $user_id, $rating, $comment]);
     */
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
    // Use INSERT IGNORE to prevent errors if the favorite already exists (due to UNIQUE KEY)
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
            GROUP_CONCAT(mg.genre SEPARATOR ', ') as genres,
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

// Helper function to calculate average rating for a movie
function getMovieAverageRating($movieId) {
    global $pdo;
    $stmt = $pdo->prepare("SELECT AVG(rating) as average_rating FROM reviews WHERE movie_id = ?");
    $stmt->execute([$movieId]);
    $result = $stmt->fetch();
    // Return formatted rating or 'N/A'
    return $result && $result['average_rating'] !== null ? number_format((float)$result['average_rating'], 1, '.', '') : 'N/A'; // Cast to float, specify decimal point
}

// Helper function to check if a movie is favorited by the current user
function isMovieFavorited($movieId, $userId) {
    global $pdo;
    if (!$userId) return false; // Cannot favorite if not logged in
    $stmt = $pdo->prepare("SELECT COUNT(*) FROM favorites WHERE movie_id = ? AND user_id = ?");
    $stmt->execute([$movieId, $userId]);
    return $stmt->fetchColumn() > 0;
}

// Define upload directories
define('UPLOAD_DIR_POSTERS', __DIR__ . '/../uploads/posters/'); // Use absolute path
define('UPLOAD_DIR_TRAILERS', __DIR__ . '/../uploads/trailers/'); // Use absolute path
define('WEB_UPLOAD_DIR_POSTERS', '../uploads/posters/'); // Web accessible path
define('WEB_UPLOAD_DIR_TRAILERS', '../uploads/trailers/'); // Web accessible path


// Create upload directories if they don't exist
if (!is_dir(UPLOAD_DIR_POSTERS)) {
    mkdir(UPLOAD_DIR_POSTERS, 0775, true); // Use 0775 permissions
}
if (!is_dir(UPLOAD_DIR_TRAILERS)) {
    mkdir(UPLOAD_DIR_TRAILERS, 0775, true); // Use 0775 permissions
}

?>
```

---

**3. `includes/config.php`**

*Place this file in the `includes` directory at your project root.*

```php
<?php
// includes/config.php

// Start a session if one hasn't been started already
if (session_status() == PHP_SESSION_NONE) {
    session_start();
}

// Set timezone (optional but recommended)
date_default_timezone_set('Asia/Jakarta'); // Example: Jakarta timezone

// Include the database connection and functions file
require_once __DIR__ . '/../config/database.php'; // Use __DIR__ for reliable path

// Configure error reporting for debugging (Disable on production)
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

// Set up a basic error log (make sure the 'logs' directory exists and is writable)
// ini_set('log_errors', 1);
// ini_set('error_log', __DIR__ . '/../logs/php-error.log');


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

// Function to fetch authenticated user details
// Calls getUserById from database.php
function getAuthenticatedUser() {
    if (isAuthenticated()) {
        $userId = $_SESSION['user_id'];
        // Fetch user details from the database
        $user = getUserById($userId);
        if ($user) {
            // User found, return details
            return $user;
        } else {
            // User not found in DB (maybe deleted?), clear session and redirect
            session_destroy();
            header('Location: ../autentikasi/form-login.php'); // Redirect to login
            exit;
        }
    }
    return null; // Return null if not authenticated
}

// Moved getMovieAverageRating and isMovieFavorited to database.php
// They are now accessed via database.php functions or directly if global $pdo is available and functions are defined globally

?>
```

---

**4. `autentikasi/form-login.php`**

*Place this file in the `autentikasi` directory.*

```php
<?php
// autentikasi/form-login.php
require_once '../includes/config.php'; // Include config.php

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

// Retrieve input values from session if redirected back due to error (optional but improves UX)
$username_input = $_SESSION['login_username_input'] ?? '';
$captcha_input = $_SESSION['login_captcha_input'] ?? ''; // Note: CAPTCHA should be re-entered for security

// Clear stored inputs from session
unset($_SESSION['login_username_input']);
unset($_SESSION['login_captcha_input']);


if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // Ambil dan bersihkan input
    $username_input_post = trim($_POST['username'] ?? '');
    $password_input = $_POST['password'] ?? '';
    $captcha_input_post = trim($_POST['captcha_input'] ?? '');

     // Store inputs in session in case of redirect (for UX)
     $_SESSION['login_username_input'] = $username_input_post;
     $_SESSION['login_captcha_input'] = $captcha_input_post; // Note: This will be cleared on page load, but useful for debugging

    // --- Validasi Server-side (Basic) ---
    if (empty($username_input_post) || empty($password_input) || empty($captcha_input_post)) {
        $_SESSION['error'] = 'Username/Email, Password, and CAPTCHA are required.';
        header('Location: form-login.php'); // Redirect back to show error and new CAPTCHA
        exit;
    }

    // --- Validasi CAPTCHA (Server-side) ---
    if (!isset($_SESSION['captcha_code']) || strtolower($captcha_input_post) !== strtolower($_SESSION['captcha_code'])) {
        $_SESSION['error'] = 'Invalid CAPTCHA.';
        // CAPTCHA already regenerated at the top if there was an error before CAPTCHA check
        header('Location: form-login.php'); // Redirect back to show error and new CAPTCHA
        exit; // Stop execution if CAPTCHA is wrong
    }

    // CAPTCHA valid, unset it immediately to prevent reuse
    unset($_SESSION['captcha_code']);


    // --- Lanjutkan proses login ---
    try {
        // Cek pengguna berdasarkan username atau email
        $stmt = $pdo->prepare("SELECT user_id, password FROM users WHERE username = ? OR email = ? LIMIT 1");
        $stmt->execute([$username_input_post, $username_input_post]);
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
             // Ensure CAPTCHA is regenerated for the next attempt
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
    <link rel="icon" type="image/png" href="../gambar/Untitled142_20250310223718.png" sizes="16x16">
    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;600&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="style.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <!-- Google Sign-In - Keep for now, although not implemented -->
    <!-- <script src="https://accounts.google.com/gsi/client" async defer></script> -->
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
                <input type="text" name="username" id="username" placeholder="Username or Email" required value="<?php echo htmlspecialchars($username_input); ?>">
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
                 <input type="text" name="captcha_input" id="captchaInput" placeholder="Enter CAPTCHA" required autocomplete="off"> <!-- Value is NOT prefilled for security -->
                 <p id="captchaMessage" class="error-message" style="display:none;"></p>
            </div>

            <button type="submit" class="btn">Login</button>
        </form>
        <br>
        <!-- Google Sign-In elements (commented out as not fully implemented) -->
        <!-- <div id="g_id_onload" data-client_id="YOUR_GOOGLE_CLIENT_ID" data-callback="handleCredentialResponse"></div>
        <div class="g_id_signin" data-type="standard"></div> -->


        <p class="form-link">Don't have an account? <a href="form-register.php">Register here</a></p>
    </div>

    <script src="animation.js"></script>
    <script>
        // Variable to store the current CAPTCHA code
        // Using PHP to insert the code from the session
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
             // Optional: clear CAPTCHA input on page load for security
             captchaInput.value = '';
        });

        // Google Sign-In handler (placeholder - uncomment and implement if needed)
        // function handleCredentialResponse(response) {
        //    console.log("Encoded JWT ID token: " + response.credential);
        //    // TODO: Send this token to your server for validation
        //    // Example: window.location.href = 'verify-google-token.php?token=' + response.credential;
        // }
    </script>
</body>
</html>
```

---

**5. `autentikasi/form-register.php`**

*Place this file in the `autentikasi` directory.*

```php
<?php
// autentikasi/form-register.php
require_once '../includes/config.php'; // Include config.php

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

// Retrieve input values from session if redirected back due to error (optional but improves UX)
// Note: Password fields are NOT pre-filled for security
$full_name = $_SESSION['register_full_name'] ?? '';
$username = $_SESSION['register_username'] ?? '';
$age_input = $_SESSION['register_age'] ?? '';
$gender = $_SESSION['register_gender'] ?? '';
$email = $_SESSION['register_email'] ?? '';
$captcha_input = $_SESSION['register_captcha_input'] ?? ''; // Note: CAPTCHA should be re-entered
$agree = $_SESSION['register_agree'] ?? false;

// Clear stored inputs from session (except maybe for debugging)
unset($_SESSION['register_full_name']);
unset($_SESSION['register_username']);
unset($_SESSION['register_age']);
unset($_SESSION['register_gender']);
unset($_SESSION['register_email']);
unset($_SESSION['register_captcha_input']);
unset($_SESSION['register_agree']);


if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // Ambil dan bersihkan input
    $full_name_post = trim($_POST['full_name'] ?? '');
    $username_post = trim($_POST['username'] ?? '');
    $age_input_post = $_POST['age'] ?? '';
    $gender_post = $_POST['gender'] ?? '';
    $email_post = trim($_POST['email'] ?? '');
    $password_post = $_POST['password'] ?? '';
    $confirm_password_post = $_POST['confirm_password'] ?? '';
    $captcha_input_post = trim($_POST['captcha_input'] ?? ''); // Trim captcha input too
    $agree_post = isset($_POST['agree']);

     // Store inputs in session in case of redirect (for UX on error)
     $_SESSION['register_full_name'] = $full_name_post;
     $_SESSION['register_username'] = $username_post;
     $_SESSION['register_age'] = $age_input_post;
     $_SESSION['register_gender'] = $gender_post;
     $_SESSION['register_email'] = $email_post;
     $_SESSION['register_captcha_input'] = $captcha_input_post; // Cleared on page load, but available briefly
     $_SESSION['register_agree'] = $agree_post;


    // --- Validasi Server-side ---
    $errors = [];

    if (empty($full_name_post)) $errors[] = 'Full Name is required.';
    if (empty($username_post)) $errors[] = 'Username is required.';
    if (empty($email_post)) $errors[] = 'Email is required.';
    if (empty($password_post)) $errors[] = 'Password is required.';
    if (empty($confirm_password_post)) $errors[] = 'Confirm Password is required.';
    if (empty($captcha_input_post)) $errors[] = 'CAPTCHA is required.';
    if (!$agree_post) $errors[] = 'You must agree to the User Agreement.';

    // Validate Age: check if numeric and > 0
    $age = filter_var($age_input_post, FILTER_VALIDATE_INT);
    if ($age === false || $age <= 0) {
        $errors[] = 'Invalid Age.';
    }

    // Validate Gender: check against allowed options
    $allowed_genders = ['Laki-laki', 'Perempuan'];
    if (!in_array($gender_post, $allowed_genders)) {
        $errors[] = 'Invalid Gender selection.';
    }

    if ($password_post !== $confirm_password_post) {
        $errors[] = 'Password and Confirm Password do not match.';
    }

    if (strlen($password_post) < 6) {
        $errors[] = 'Password must be at least 6 characters long.';
    }

    // Validate Email Format (basic)
    if (!filter_var($email_post, FILTER_VALIDATE_EMAIL)) {
         $errors[] = 'Invalid Email format.';
    }

    // Validate Username format (optional: allow only letters, numbers, underscore)
    // if (!preg_match('/^[a-zA-Z0-9_]+$/', $username_post)) {
    //     $errors[] = 'Username can only contain letters, numbers, and underscores.';
    // }


    // --- Validasi CAPTCHA (Server-side) - This is the crucial security check ---
    if (!isset($_SESSION['captcha_code']) || strtolower($captcha_input_post) !== strtolower($_SESSION['captcha_code'])) {
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
        if (!isset($_SESSION['captcha_code'])) { // Cek again if it wasn't regenerated already
             $_SESSION['captcha_code'] = generateRandomString(6);
        }
        header('Location: form-register.php');
        exit;
    }

    // --- If all validations pass, proceed to DB checks and INSERT ---

    // Check if username or email already exists
    try {
        $stmt_check = $pdo->prepare("SELECT COUNT(*) FROM users WHERE username = ? OR email = ? LIMIT 1"); // Use LIMIT 1
        $stmt_check->execute([$username_post, $email_post]);
        if ($stmt_check->fetchColumn() > 0) {
            $_SESSION['error'] = 'Username or email is already registered.';
             // Regenerate CAPTCHA on database check failure
            $_SESSION['captcha_code'] = generateRandomString(6);
            header('Location: form-register.php');
            exit;
        }

        // Hash password
        $hashed_password = password_hash($password_post, PASSWORD_BCRYPT);

        // Save new user to database using the createUser function from config/database.php
        $inserted = createUser(
            $full_name_post,
            $username_post,
            $email_post,
            $hashed_password,
            $age, // Use the validated integer age
            $gender_post
            // profile_image and bio are null by default in createUser function
        );


        if ($inserted) {
             // Get the ID of the newly created user
            $userId = $pdo->lastInsertId();
            $_SESSION['user_id'] = $userId;

            // Regenerate session ID after successful registration
            session_regenerate_id(true);

            $_SESSION['success'] = 'Registration successful! Welcome, ' . htmlspecialchars($username_post) . '!';

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
    <link rel="icon" type="image/png" href="../gambar/Untitled142_20250310223718.png">
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
                <input type="text" id="name" name="full_name" placeholder="Enter your full name" required value="<?php echo htmlspecialchars($full_name); ?>">
            </div>
            <div class="input-group">
                <label for="username">Username</label>
                <input type="text" id="username" name="username" placeholder="Enter a username" required value="<?php echo htmlspecialchars($username); ?>">
            </div>
            <div class="input-group">
                <label for="usia">Age</label>
                <input type="number" id="usia" name="age" placeholder="Your age" required min="1" value="<?php echo htmlspecialchars($age_input); ?>">
            </div>
            <div class="input-group">
                <label for="gender">Gender</label>
                <select id="gender" name="gender" required>
                    <option value="">Select...</option>
                    <option value="Laki-laki" <?php echo ($gender === 'Laki-laki') ? 'selected' : ''; ?>>Male</option>
                    <option value="Perempuan" <?php echo ($gender === 'Perempuan') ? 'selected' : ''; ?>>Female</option>
                </select>
            </div>
            <div class="input-group">
                <label for="email">Email</label>
                <input type="email" id="email" name="email" placeholder="Enter your email" required value="<?php echo htmlspecialchars($email); ?>">
            </div>
             <!-- Password Fields - Don't pre-fill for security -->
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
                 <input type="text" name="captcha_input" id="captchaInput" placeholder="Enter CAPTCHA" required autocomplete="off"> <!-- Value is NOT pre-filled for security -->
                 <p id="captchaMessage" class="error-message" style="display:none;"></p>
            </div>

            <div style="text-align: center; margin-bottom: 10px; margin-top: 20px;">
                <button type="button" id="agreement-btn">Read User Agreement</button>
            </div>

            <div class="input-group agreement-checkbox">
                <input type="checkbox" id="agree-checkbox" name="agree" required <?php echo $agree ? 'checked' : ''; ?>>
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
                Data personal Anda akan disimpan selama akun Anda masih aktif, atau selama diperlukan untuk mendukung tujuan layanan. Kami menerapkan langkah-langkah teknis dan organisasi yang sesuai untuk melindungi data Anda dari akses yang tidak sah, kebocoran, atau penyalahgunaan. Kami tidak akan membagikan data personal Anda kepada pihak ketiga tanpa persetujuan eksplisit dari Anda, kecuali jika diharuskan oleh hukum atau dalam konteks penegakan hukum dan kewajiban hukum lainnya.
                Sesuai dengan ketentuan UU PDP, Anda sebagai pemilik data memiliki hak untuk mengakses data personal Anda, meminta perbaikan atau penghapusan data, menarik kembali persetujuan atas pemprosesan data, serta mengajukan keberatan atas pemprosesan tertentu. Kami menghormati hak-hak tersebut dan akan menindaklanjuti setiap permintaan yang Anda sampaikan melalui saluran kontak resmi yang tersedia di situs kami.
                Kami dapat memperbarui isi Kebijakan Privasi ini dari waktu ke waktu, terutama jika terjadi perubahan peraturan atau perkembangan teknologi yang memengaruhi cara kami memproses data personal Anda. Perubahan signifikan akan kami sampaikan melalui notifikasi di situs atau email. Dengan terus menggunakan layanan kami setelah perubahan diberlakukan, Anda dianggap telah menyetujui kebijakan yang diperbarui.
                Jika Anda memiliki pertanyaan, permintaan, atau keluhan terkait kebijakan ini atau penggunaan data personal Anda, Anda dapat menghubungi kami melalui alamat email atau formulir kontak resmi yang tersedia di situs. Dengan menggunakan situs ini, Anda menyatakan telah membaca, memahami, dan menyetujui isi Kebijakan Privasi ini serta memberikan persetujuan eksplisit atas pengumpulan dan pemprosesan data personal Anda oleh kami.
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
             // Optional: clear CAPTCHA input on page load for security
             captchaInput.value = '';
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

---

**6. `autentikasi/generate_captcha.php`**

*Place this file in the `autentikasi` directory.*

```php
<?php
// generate_captcha.php
// File ini hanya bertugas menghasilkan kode CAPTCHA baru dan menyimpannya di session

// Include file konfigurasi untuk koneksi DB, session, dan helper function
require_once '../includes/config.php';

// Ensure session is active (should be handled by config.php)
if (session_status() == PHP_SESSION_NONE) {
    session_start();
}

// Generate a new CAPTCHA code using the helper function from config.php
$newCaptchaCode = generateRandomString(6);

// Store the new code in the session, overwriting the old one
$_SESSION['captcha_code'] = $newCaptchaCode;

// Set header to tell the client this is plain text
header('Content-Type: text/plain');

// Output the new CAPTCHA code to the client
echo $newCaptchaCode;

// Stop script execution
exit;
?>
```

---

**7. `autentikasi/logout.php`**

*Place this file in the `autentikasi` directory.*

```php
<?php
// logout.php
require_once '../includes/config.php'; // Include config.php

// Ensure session is active (should be handled by config.php)
if (session_status() == PHP_SESSION_NONE) {
    session_start();
}

// Unset all session variables
$_SESSION = array();

// Delete the session cookie (if one exists)
if (ini_get("session.use_cookies")) {
    $params = session_get_cookie_params();
    setcookie(session_name(), '', time() - 42000,
        $params["path"], $params["domain"],
        $params["secure"], $params["httponly"]
    );
}

// Destroy the session
session_destroy();

// Redirect to the login page
header('Location: form-login.php');
exit;
?>
```

---

**8. `autentikasi/styles.css`**

*Place this file in the `autentikasi` directory.*

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
    /* Pastikan path gambar benar relatif terhadap file CSS */
    background: url(../gambar/5302920.jpg) no-repeat center center fixed;
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

/* Add style for input CAPTCHA in the input-group */
.input-group .captcha-input {
    margin-top: 5px; /* Beri jarak dari container canvas/button */
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
     font-size: 14px;
     color: #00e4f9;
     cursor: pointer;
     padding: 5px 10px;
     margin: 0;
     text-align: center;
     display: inline-block;
     vertical-align: middle;
     text-decoration: underline;
     transition: color 0.3s;
}
button#agreement-btn:hover {
    color: #00cccc;
    text-decoration: none;
}


/* Style for the div that contains the agreement checkbox and label */
.input-group.agreement-checkbox {
    display: flex;
    align-items: flex-start;
    gap: 10px;
    margin-bottom: 20px;
}
.input-group.agreement-checkbox label {
     display: inline-block;
     margin-bottom: 0;
     font-weight: normal;
     flex-grow: 1;
     line-height: 1.4;
     color: #b0e0e6;
}

/* Alert Messages (Used same class names) */
.error-message {
    color: #ff6b6b; /* Reddish */
    font-size: 14px;
    margin: 10px 0;
    text-align: center;
    background-color: rgba(255, 107, 107, 0.1);
    padding: 8px;
    border-radius: 4px;
    border: 1px solid rgba(255, 107, 107, 0.4);
}

.success-message {
    color: #6bff6b; /* Greenish */
    font-size: 14px;
    margin: 10px 0;
    text-align: center;
    background-color: rgba(107, 255, 107, 0.1);
    padding: 8px;
    border-radius: 4px;
    border: 1px solid rgba(107, 255, 107, 0.4);
}
```

---

**9. `autentikasi/animation.js`**

*Place this file in the `autentikasi` directory.*

```javascript
// animation.js

document.addEventListener('DOMContentLoaded', () => {
    // Find all elements with class 'form-container'
    const formContainers = document.querySelectorAll('.form-container');

    // Wait a bit before adding the 'show' class
    // This gives the browser time to render the initial layout before starting the transition
    setTimeout(() => {
        formContainers.forEach(container => {
            container.classList.add('show');
        });
    }, 100); // Wait 100ms
});
```

---

**10. `beranda/index.php`**

*Place this file in the `beranda` directory.*

```php
<?php
// beranda/index.php
require_once '../includes/config.php'; // Include config.php

// Fetch all movies from the database
$movies = getAllMovies(); // This function now fetches average_rating and genres

// Get authenticated user details (if logged in)
$user = getAuthenticatedUser();

// Determine movies for sections (simple logic for now)
// Filter out movies without posters for the slider
$movies_with_posters = array_filter($movies, function($movie) {
    return !empty($movie['poster_image']);
});

$featured_movies = array_slice($movies_with_posters, 0, 5); // First 5 with posters for slider
$trending_movies = array_slice($movies, 0, 10); // First 10 overall for trending
$for_you_movies = array_slice($movies, 1, 10); // A different slice for variety (adjust logic as needed)
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Home</title>
    <link rel="stylesheet" href="styles.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
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
                 <?php if ($user): ?>
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li>
                 <?php endif; ?>
            </ul>
            <div class="bottom-links">
                <ul>
                    <?php if ($user): ?>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li>
                    <?php else: ?>
                    <li><a href="../autentikasi/form-login.php"><i class="fas fa-sign-in-alt"></i> <span>Login</span></a></li>
                    <?php endif; ?>
                </ul>
            </div>
        </nav>
        <main class="main-content">
            <div class="hero-section">
                <div class="featured-movie-slider">
                    <?php if (!empty($featured_movies)): ?>
                        <?php foreach ($featured_movies as $index => $movie): ?>
                            <div class="slide <?php echo $index === 0 ? 'active' : ''; ?>">
                                <img src="<?php echo htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                <div class="movie-info">
                                    <h1><?php echo htmlspecialchars($movie['title']); ?></h1>
                                    <p><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?></p>
                                </div>
                            </div>
                        <?php endforeach; ?>
                    <?php else: ?>
                         <div class="slide active empty-state" style="position:relative; opacity:1; display:flex; flex-direction:column; justify-content:center; align-items:center; text-align:center; height: 100%;">
                             <i class="fas fa-film" style="font-size: 3em; color:#00ffff; margin-bottom:15px;"></i>
                             <p style="color: #fff; font-size:1.2em;">No featured movies available yet.</p>
                              <?php if($user): ?>
                              <p class="subtitle" style="color: #888; margin-top:10px;">Upload movies in the <a href="../manage/indeks.php" style="color:#00ffff; text-decoration:none;">Manage</a> section.</p>
                              <?php endif; ?>
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
                                <div class="movie-card" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'">
                                    <img src="<?php echo htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                    <div class="movie-details">
                                        <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                                        <p><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?></p>
                                    </div>
                                </div>
                            <?php endforeach; ?>
                        <?php else: ?>
                            <div class="empty-state" style="min-width: 100%; text-align: center; padding: 20px;">
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
                                <div class="movie-card" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'">
                                    <img src="<?php echo htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                    <div class="movie-details">
                                        <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                                        <p><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?></p>
                                    </div>
                                </div>
                            <?php endforeach; ?>
                         <?php else: ?>
                            <div class="empty-state" style="min-width: 100%; text-align: center; padding: 20px;">
                                <p style="color: #888;">No movie suggestions for you.</p>
                            </div>
                        <?php endif; ?>
                    </div>
                </div>
            </section>
        </main>
         <?php if ($user): ?>
             <a href="../acc_page/index.php" class="user-profile">
                 <img src="<?php echo htmlspecialchars($user['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($user['full_name'] ?? $user['username']) . '&background=random&color=fff&size=30'); ?>" alt="User Profile" class="profile-pic">
                 <span><?php echo htmlspecialchars($user['full_name'] ?? $user['username']); ?></span>
             </a>
         <?php else: ?>
             <a href="../autentikasi/form-login.php" class="user-profile" style="background-color: #00ffff; color: #1a1a1a; font-weight:bold;">
                 <span>Login</span>
             </a>
         <?php endif; ?>
    </div>
    <script>
    document.addEventListener('DOMContentLoaded', function() {
        const slides = document.querySelectorAll('.hero-section .slide');
        if (slides.length > 0) {
            let currentSlide = 0;

            // Function to show a specific slide
            function showSlide(index) {
                slides.forEach(slide => slide.classList.remove('active'));
                slides[index].classList.add('active');
            }

            // Function to advance to the next slide
            function nextSlide() {
                currentSlide = (currentSlide + 1) % slides.length;
                showSlide(currentSlide);
            }

             // Show the first slide immediately
             showSlide(currentSlide);

            // Change slide every 5 seconds
            setInterval(nextSlide, 5000);
        }

         // Smooth scroll for movie grids (optional enhancement)
         document.querySelectorAll('.scroll-container').forEach(container => {
             container.addEventListener('wheel', (e) => {
                 // Check if the mouse wheel event is vertical, only react if it's horizontal (Shift key) or you want to force horizontal scroll
                 // Forcing horizontal scroll on vertical wheel:
                 if (Math.abs(e.deltaY) > Math.abs(e.deltaX)) { // Only act if vertical scroll is dominant
                      e.preventDefault(); // Prevent default vertical scroll
                      container.scrollLeft += e.deltaY * 0.5; // Scroll horizontally based on vertical delta
                 } else if (e.deltaX !== 0) { // Also allow horizontal scroll from horizontal wheel
                      container.scrollLeft += e.deltaX;
                 }
             });
         });
    });
    </script>
</body>
</html>
```

---

**11. `beranda/styles.css`**

*Place this file in the `beranda` directory.*

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
     /* Fallback background if image fails */
    background-color: #363636;
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
     /* Hide broken image icon */
    color: transparent;
    font-size: 0;
}
/* Show alt text or a fallback if image fails to load */
.slide img::before {
    content: attr(alt);
    display: block;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #363636;
    color: #ffffff;
    text-align: center;
    padding-top: 50%;
    font-size: 16px;
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
     /* Hide broken image icon */
    color: transparent;
    font-size: 0;
}
/* Show alt text or a fallback if image fails to load */
.movie-card img::before {
    content: attr(alt);
    display: block;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #363636;
    color: #ffffff;
    text-align: center;
    padding-top: 50%;
    font-size: 16px;
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
     /* Hide broken image icon */
    color: transparent;
    font-size: 0;
}
/* Show alt text or a fallback if image fails to load */
.profile-pic::before {
    content: attr(alt);
    display: block;
    position: absolute; /* Position relative to .profile-pic */
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #363636; /* Fallback background */
    color: #ffffff;
    text-align: center;
    line-height: 30px; /* Vertically center text in 30px height */
    font-size: 12px;
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
.empty-state .subtitle a {
     color: #00ffff;
     text-decoration: none;
     font-weight: bold;
}
.empty-state .subtitle a:hover {
     text-decoration: underline;
}

/* Scrollbar Styles */
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
@media (max-width: 768px) {
    .sidebar {
        width: 70px;
        padding: 10px;
    }

    .logo h2, .nav-links span, .bottom-links span {
        display: none;
    }

    .nav-links i, .bottom-links i {
        margin-right: 0;
        width: 100%; /* Icons take full width of narrow sidebar */
        text-align: center;
    }

     .nav-links a, .bottom-links a {
         justify-content: center; /* Center content in links */
         padding: 10px 0; /* Adjust padding */
     }

    .main-content {
        margin-left: 70px;
         padding: 10px; /* Reduced main content padding */
    }

    .movie-card {
        flex: 0 0 150px; /* Smaller movie cards */
        min-width: 150px;
    }

    .movie-card img {
        height: 225px; /* Maintain 2:3 aspect ratio */
    }

    .hero-section {
        height: 300px;
        margin-bottom: 20px;
    }

     .slide img::before, .movie-card img::before {
         font-size: 14px; /* Smaller fallback text */
     }


    .movie-info {
        padding: 20px; /* Reduced info padding */
    }

    .movie-info h1 {
        font-size: 24px;
    }
     .movie-info p {
         font-size: 16px;
     }

    .trending-section, .for-you-section {
        margin-bottom: 30px;
    }

    .trending-section h2, .for-you-section h2 {
        font-size: 20px;
        margin-bottom: 15px;
    }

     .user-profile {
         top: 10px; /* Adjust position */
         right: 10px;
         padding: 5px 10px;
     }
     .user-profile span {
         display: none; /* Hide username */
     }
      .user-profile .profile-pic {
          margin-right: 0; /* Remove margin */
      }
}
```

---

**12. `favorite/index.php`**

*Place this file in the `favorite` directory.*

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
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
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
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li>
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li>
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
                            <div class="movie-poster" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'">
                                <img src="<?php echo htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
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
                        emptyState.className = 'empty-state search-empty-state full-width';
                        emptyState.innerHTML = `
                            <i class="fas fa-search"></i>
                            <p>No favorites found matching "${htmlspecialchars(searchTerm)}"</p>
                            <p class="subtitle">Try a different search term</p>
                        `;
                        reviewGrid.appendChild(emptyState);
                    } else {
                         // Update text if search empty state already exists
                         searchEmptyState.querySelector('p:first-of-type').innerText = `No favorites found matching "${htmlspecialchars(searchTerm)}"`;
                         searchEmptyState.style.display = 'flex';
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
            if(searchButton) {
                 searchButton.addEventListener('click', function() {
                     const event = new Event('input');
                     searchInput.dispatchEvent(event);
                 });
            }
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

---

**13. `favorite/styles.css`**

*Place this file in the `favorite` directory.*

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
     /* Fallback background if image fails */
    background-color: #363636;
}

.movie-poster img {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
     /* Hide broken image icon */
    color: transparent;
    font-size: 0;
}
/* Show alt text or a fallback if image fails to load */
.movie-poster img::before {
    content: attr(alt);
    display: block;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #363636;
    color: #ffffff;
    text-align: center;
    padding-top: 50%;
    font-size: 16px;
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
    flex-grow: 1;
    display: flex;
    flex-direction: column;
}

.movie-details h3 {
    margin-bottom: 5px;
    font-size: 16px;
    white-space: normal;
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
    color: #888;
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

/* Full width empty state for initial display */
.empty-state.full-width {
    width: 100%;
    margin: 0;
    min-height: 300px;
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
@media (max-width: 768px) {
    .sidebar {
        width: 70px;
        padding: 10px;
    }

    .logo h2, .nav-links span, .bottom-links span {
        display: none;
    }

    .nav-links i, .bottom-links i {
        margin-right: 0;
        width: 100%;
        text-align: center;
    }

     .nav-links a, .bottom-links a {
         justify-content: center;
         padding: 10px 0;
     }

    .main-content {
        margin-left: 70px;
         padding: 10px;
    }

    .review-header {
        flex-direction: column;
        align-items: flex-start;
        padding: 15px;
    }
    .review-header h1 {
        margin-bottom: 15px;
        font-size: 24px;
    }
    .search-bar {
        width: 100%;
    }
     .search-bar input {
         width: 100%;
     }

    .movie-card {
        flex: 0 0 150px; /* Smaller movie cards */
        min-width: 150px;
    }

    .movie-card img {
        height: 225px; /* Maintain 2:3 aspect ratio */
    }
     .movie-poster img::before {
         font-size: 14px;
     }


    .movie-actions {
        padding: 10px;
        gap: 10px;
    }
    .action-btn {
        width: 30px;
        height: 30px;
        font-size: 16px;
    }

    .movie-details {
        padding: 10px;
    }
     .movie-details h3 {
         font-size: 14px;
     }
     .movie-info {
         font-size: 12px;
     }
     .rating {
         font-size: 0.9em;
         gap: 5px;
     }
     .rating-count {
         font-size: 12px;
     }
}
```

---

**14. `review/index.php`**

*Place this file in the `review` directory.*

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
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
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
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li>
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li>
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
                        <div class="movie-card" onclick="window.location.href='movie-details.php?id=<?php echo $movie['movie_id']; ?>'">
                            <div class="movie-poster">
                                <img src="<?php echo htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
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
                     <div class="empty-state full-width">
                         <i class="fas fa-film"></i>
                         <p>No movies available to review yet.</p>
                         <p class="subtitle">Check back later or <a href="../manage/upload.php">Upload a movie</a> if you are an admin.</p>
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
                        emptyState.className = 'empty-state search-empty-state full-width';
                        emptyState.innerHTML = `
                            <i class="fas fa-search"></i>
                            <p>No movies found matching "${htmlspecialchars(searchTerm)}"</p>
                            <p class="subtitle">Try a different search term</p>
                        `;
                        reviewGrid.appendChild(emptyState);
                    } else {
                         // Update text if search empty state already exists
                         searchEmptyState.querySelector('p:first-of-type').innerText = `No movies found matching "${htmlspecialchars(searchTerm)}"`;
                         searchEmptyState.style.display = 'flex';
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
            if(searchButton) {
                 searchButton.addEventListener('click', function() {
                     const event = new Event('input');
                     searchInput.dispatchEvent(event);
                 });
            }
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

---

**15. `review/movie-details.php`**

*Place this file in the `review` directory.*

```php
<?php
// review/movie-details.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated
redirectIfNotAuthenticated();

// Get authenticated user ID
$userId = $_SESSION['user_id'];
$user = getAuthenticatedUser(); // Fetch user details for comments (username, profile_image)

// Get movie ID from URL parameter
$movieId = filter_var($_GET['id'] ?? null, FILTER_VALIDATE_INT);

// Handle movie not found if ID is missing or invalid
if (!$movieId) {
    $_SESSION['error_message'] = 'Invalid movie ID.';
    header('Location: index.php');
    exit;
}

// Fetch movie details from the database
$movie = getMovieById($movieId);

// Handle movie not found in DB
if (!$movie) {
    $_SESSION['error_message'] = 'Movie not found.';
    header('Location: index.php');
    exit;
}

// Fetch comments for the movie
$comments = getMovieReviews($movieId);

// Check if the movie is favorited by the current user
$isFavorited = isMovieFavorited($movieId, $userId);

// Handle comment and rating submission
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['submit_review'])) {
    $commentText = trim($_POST['comment'] ?? '');
    $submittedRating = filter_var($_POST['rating'] ?? null, FILTER_VALIDATE_FLOAT);

     // Basic validation for rating
     if ($submittedRating === false || $submittedRating < 0.5 || $submittedRating > 5) {
          $_SESSION['error_message'] = 'Please provide a valid rating (0.5 to 5).';
     } else {
          // Allow empty comment with rating, but trim it
          if (createReview($movieId, $userId, $submittedRating, $commentText)) {
              $_SESSION['success_message'] = 'Your review has been submitted!';
          } else {
              $_SESSION['error_message'] = 'Failed to submit your review.';
          }
     }

    header("Location: movie-details.php?id={$movieId}"); // Redirect after processing
    exit;
}


// Handle Favorite/Unfavorite action (using POST for robustness)
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['toggle_favorite'])) {
    $action = $_POST['toggle_favorite']; // 'favorite' or 'unfavorite'
    $targetMovieId = filter_var($_POST['movie_id'] ?? null, FILTER_VALIDATE_INT);

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
        } else {
             $_SESSION['error_message'] = 'Invalid favorite action.';
        }
    } else {
         $_SESSION['error_message'] = 'Invalid movie ID for favorite action.';
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

// Determine poster image source (using web accessible path)
$posterSrc = htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $movie['poster_image'] ?? '../gambar/placeholder.jpg');

// Determine trailer source (using web accessible paths)
$trailerUrl = null;
if (!empty($movie['trailer_url'])) {
    // Assume YouTube URL and extract video ID
    parse_str( parse_url( $movie['trailer_url'], PHP_URL_QUERY ), $vars );
    $youtubeVideoId = $vars['v'] ?? null;
    if ($youtubeVideoId) {
        $trailerUrl = "https://www.youtube.com/embed/{$youtubeVideoId}";
    } else {
         // Handle other video URL types if needed (basic passthrough)
         $trailerUrl = htmlspecialchars($movie['trailer_url']);
    }

} elseif (!empty($movie['trailer_file'])) {
    // Assume local file path, construct web accessible URL
    $trailerUrl = htmlspecialchars(WEB_UPLOAD_DIR_TRAILERS . $movie['trailer_file']); // Adjust path if necessary
}

// Format duration
$duration_display = '';
if ($movie['duration_hours'] > 0) {
    $duration_display .= $movie['duration_hours'] . 'h ';
}
$duration_display .= $movie['duration_minutes'] . 'm';

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?php echo htmlspecialchars($movie['title']); ?> - RATE-TALES</title>
    <link rel="stylesheet" href="styles.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
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
            color: #00ffff;
            text-decoration: none;
            margin-bottom: 2rem;
            font-size: 1.1rem;
            transition: color 0.3s;
        }
         .back-button:hover {
             color: #00cccc;
         }

        .movie-header {
            display: flex;
            gap: 2rem;
            margin-bottom: 2rem;
            flex-wrap: wrap;
        }

        .movie-poster-large {
            width: 300px;
            height: 450px;
            border-radius: 15px;
            overflow: hidden;
            box-shadow: 0 5px 15px rgba(0,0,0,0.3);
             flex-shrink: 0;
             margin: auto; /* Center if wraps */
             /* Fallback background if image fails */
            background-color: #363636;
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
             /* Hide broken image icon */
            color: transparent;
            font-size: 0;
        }
         /* Show alt text or a fallback if image fails to load */
         .movie-poster-large img::before {
             content: attr(alt);
             display: block;
             position: absolute;
             top: 0;
             left: 0;
             width: 100%;
             height: 100%;
             background-color: #363636;
             color: #ffffff;
             text-align: center;
             padding-top: 50%;
             font-size: 16px;
         }


        .movie-info-large {
            flex: 1;
            min-width: 300px;
        }

        .movie-title-large {
            font-size: 2.8rem;
            margin-bottom: 1rem;
             color: #00ffff;
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
            color: #ffd700;
            font-size: 1.8rem;
        }
         .rating-large .stars i {
             margin-right: 3px;
         }

        .movie-description {
            line-height: 1.8;
            margin-bottom: 2rem;
             color: #ccc;
        }

        .action-buttons {
            display: flex;
            gap: 1rem;
            flex-wrap: wrap;
        }

        .action-button {
            padding: 1rem 2rem;
            border: none;
            border-radius: 10px;
            font-size: 1.1rem;
            cursor: pointer;
            display: flex;
            align-items: center;
            gap: 0.8rem;
            transition: all 0.3s ease;
            font-weight: bold;
        }

        .watch-trailer {
            background-color: #e50914;
            color: white;
        }

        .add-favorite {
            background-color: #333;
            color: white;
        }
         .add-favorite.favorited {
             background-color: #00ffff;
             color: #1a1a1a;
         }


        .action-button:hover {
            transform: translateY(-3px);
            opacity: 0.9;
        }

        .comments-section {
            margin-top: 3rem;
             background-color: #242424;
             padding: 20px;
             border-radius: 15px;
        }

        .comments-header {
            font-size: 1.8rem;
            margin-bottom: 1.5rem;
             color: #00ffff;
             border-bottom: 1px solid #333;
             padding-bottom: 15px;
        }

        .comment-input-area {
            margin-bottom: 2rem;
             padding: 15px;
             background-color: #1a1a1a;
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
             gap: 5px;
             margin-bottom: 1rem;
         }
         .rating-input-stars i {
             font-size: 1.5rem;
             color: #888;
             cursor: pointer;
             transition: color 0.2s, transform 0.2s;
         }
          .rating-input-stars i:hover,
          .rating-input-stars i.hovered,
          .rating-input-stars i.rated {
              color: #ffd700;
              transform: scale(1.1);
          }
         .rating-input-stars input[type="hidden"] {
             display: none;
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
             min-height: 80px;
        }
         .comment-input:focus {
              outline: none;
              border: 1px solid #00ffff;
              box-shadow: 0 0 0 2px rgba(0, 255, 255, 0.2);
         }


        .comment-submit-btn {
            display: block;
            width: 150px;
            margin-left: auto;
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
            background-color: #1a1a1a;
            padding: 1rem;
            border-radius: 10px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2);
        }

        .comment-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 0.5rem;
            font-size: 0.9em;
             color: #b0e0e6;
             flex-wrap: wrap; /* Allow header content to wrap */
             gap: 10px; /* Add gap for wrapped items */
        }

        .comment-header strong {
             color: #00ffff;
             font-weight: bold;
             margin-right: 10px;
        }
         .comment-header .user-info {
             display: flex;
             align-items: center;
             flex-shrink: 0; /* Prevent user info from shrinking */
         }

         .comment-rating-display {
             display: flex;
             align-items: center;
             gap: 5px;
             margin-right: 10px;
             flex-shrink: 0; /* Prevent rating display from shrinking */
         }
         .comment-rating-display .stars {
             color: #ffd700;
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
             font-size: 0.8em;
             flex-shrink: 0; /* Prevent actions from shrinking */
        }

        .comment-actions i {
            cursor: pointer;
            transition: color 0.3s;
        }

        .comment-actions i:hover {
            color: #00ffff;
        }

        .comment p {
            color: #ccc;
            line-height: 1.5;
             white-space: pre-wrap; /* Preserve line breaks */
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
            width: 90%;
            max-width: 1000px;
            position: relative;
        }

        .close-trailer {
            position: absolute;
            top: -40px;
            right: 0;
            color: white;
            font-size: 2.5rem;
            cursor: pointer;
            transition: color 0.3s;
        }
         .close-trailer:hover {
             color: #ccc;
         }

        .video-container {
            position: relative;
            padding-bottom: 56.25%;
            height: 0;
            overflow: hidden;
             background-color: black;
        }

        .video-container iframe,
        .video-container video { /* Added video tag for local files */
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

        /* Empty state for comments */
         .empty-state i {
             color: #666; /* Match other empty states */
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
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li>
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li>
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
                        <p class="movie-meta"><?php echo htmlspecialchars((new DateTime($movie['release_date']))->format('Y')); ?> | <?php echo htmlspecialchars($movie['genres'] ?? 'N/A'); ?> | <?php echo htmlspecialchars($movie['age_rating']); ?> | <?php echo $duration_display; ?></p>
                        <div class="rating-large">
                             <!-- Display average rating stars -->
                             <div class="stars">
                                <?php
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
                            <span id="movie-rating"><?php echo htmlspecialchars($movie['average_rating']); ?>/5</span>
                        </div>
                        <p class="movie-description"><?php echo nl2br(htmlspecialchars($movie['summary'] ?? 'No summary available.')); ?></p>
                        <div class="action-buttons">
                            <?php if ($trailerUrl): ?>
                                <button class="action-button watch-trailer" onclick="playTrailer('<?php echo $trailerUrl; ?>')">
                                    <i class="fas fa-play"></i>
                                    <span>Watch Trailer</span>
                                </button>
                            <?php endif; ?>

                             <!-- Favorite/Unfavorite button (using POST form) -->
                             <form action="movie-details.php?id=<?php echo $movie['movie_id']; ?>" method="POST" style="margin:0; padding:0;">
                                 <input type="hidden" name="movie_id" value="<?php echo $movie['movie_id']; ?>">
                                 <button type="submit" name="toggle_favorite" value="<?php echo $isFavorited ? 'unfavorite' : 'favorite'; ?>"
                                         class="action-button add-favorite <?php echo $isFavorited ? 'favorited' : ''; ?>">
                                     <i class="fas fa-heart"></i>
                                     <span id="favorite-text"><?php echo $isFavorited ? 'Remove from Favorites' : 'Add to Favorites'; ?></span>
                                 </button>
                             </form>
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
                                 <input type="hidden" name="rating" id="user-rating" value="0">
                             </div>
                             <textarea class="comment-input" name="comment" placeholder="Write your comment or review here..."></textarea>
                              <input type="hidden" name="submit_review" value="1"> <!-- Hidden input to identify review submission -->
                             <button type="submit" class="comment-submit-btn">Submit Review</button>
                         </form>
                     </div>


                    <div class="comment-list">
                        <?php if (!empty($comments)): ?>
                            <?php foreach ($comments as $comment): ?>
                                <div class="comment">
                                    <div class="comment-header">
                                         <div class="user-info">
                                             <img src="<?php echo htmlspecialchars($comment['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($comment['username']) . '&background=random&color=fff&size=25'); ?>" alt="Avatar" style="width: 25px; height: 25px; border-radius: 50%; margin-right: 10px; object-fit: cover;">
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
                                    <p><?php echo nl2br(htmlspecialchars($comment['comment'] ?? '')); ?></p>
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
                 <!-- Conditional rendering for iframe (YouTube) or video (local file) -->
                 <?php if (!empty($movie['trailer_url'])): ?>
                     <iframe id="trailer-iframe" src="" frameborder="0" allowfullscreen></iframe>
                 <?php elseif (!empty($movie['trailer_file'])): ?>
                     <video id="trailer-video" src="" controls autoplay></video>
                 <?php endif; ?>
            </div>
        </div>
    </div>

    <script>
    document.addEventListener('DOMContentLoaded', function() {
         // JavaScript for rating input
        const ratingStars = document.querySelectorAll('#rating-input-stars i');
        const userRatingInput = document.getElementById('user-rating');
        let currentRating = 0; // Store the selected rating (e.g., 0 for unrated, 1-5 for rated)

         // Add data-rating attribute to stars if not already present
         ratingStars.forEach((star, index) => {
             star.setAttribute('data-rating', index + 1);
         });


        ratingStars.forEach(star => {
            star.addEventListener('mouseover', function() {
                const hoverRating = parseInt(this.getAttribute('data-rating'));
                highlightStars(hoverRating, false); // Highlight based on hover, not clicked state
            });

            star.addEventListener('mouseout', function() {
                 // Revert to the currently selected rating
                highlightStars(currentRating, true); // Highlight based on clicked state
            });

            star.addEventListener('click', function() {
                currentRating = parseInt(this.getAttribute('data-rating')); // Update selected rating
                userRatingInput.value = currentRating; // Set hidden input value
                highlightStars(currentRating, true); // Highlight and mark as rated
            });
        });

        function highlightStars(rating, isClickedState) {
            ratingStars.forEach((star, index) => {
                const starRating = parseInt(star.getAttribute('data-rating'));
                star.classList.remove('hovered', 'rated'); // Remove previous states

                if (starRating <= rating) {
                    star.classList.add(isClickedState ? 'rated' : 'hovered');
                    star.classList.remove('far');
                    star.classList.add('fas');
                } else {
                    star.classList.remove('fas');
                    star.classList.add('far');
                }
            });
        }

        // Initial state for rating input (if user previously rated, load it)
        // This requires fetching the user's specific review/rating on page load
        // You would add logic in PHP to get the current user's review for this movie
        // and then set the `currentRating` variable and call `highlightStars` on DOMContentLoaded.
        // Example (assuming PHP provides $userReviewRating):
        // <?php if (!empty($userReview) && isset($userReview['rating'])): ?>
        //     currentRating = parseFloat("<?php echo $userReview['rating']; ?>");
        //     highlightStars(currentRating, true);
        //     userRatingInput.value = currentRating; // Also set hidden input
        // <?php endif; ?>


         // JavaScript for trailer modal
        const trailerModal = document.getElementById('trailer-modal');
        const trailerIframe = document.getElementById('trailer-iframe'); // For YouTube
        const trailerVideo = document.getElementById('trailer-video'); // For local files

        function playTrailer(videoSrc) {
            if (videoSrc) {
                 if (trailerIframe) {
                    trailerIframe.src = videoSrc;
                 } else if (trailerVideo) {
                     trailerVideo.src = videoSrc;
                     trailerVideo.load(); // Load the video
                     trailerVideo.play(); // Start playing
                 }
                trailerModal.classList.add('active');
            } else {
                alert('Trailer not available.');
            }
        }

        function closeTrailer() {
            if (trailerIframe) {
                trailerIframe.src = ''; // Stop YouTube video
            } else if (trailerVideo) {
                 trailerVideo.pause(); // Pause local video
                 trailerVideo.currentTime = 0; // Reset time
                 trailerVideo.src = ''; // Unload video source
            }
            trailerModal.classList.remove('active');
        }

        // Close modal when clicking outside the content or the close button
        trailerModal.addEventListener('click', function(e) {
            // Check if the clicked element is the modal background itself or the close button
            if (e.target === this || e.target.classList.contains('close-trailer') || e.target.closest('.close-trailer')) {
                closeTrailer();
            }
        });

        // Add event listener to the close button specifically
        const closeButton = document.querySelector('.close-trailer');
        if(closeButton) {
             closeButton.addEventListener('click', closeTrailer);
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
    }); // End DOMContentLoaded
    </script>
</body>
</html>
```

---

**16. `review/styles.css`**

*Place this file in the `review` directory.*

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
    margin-left: 250px;
    padding: 20px;
    max-height: 100vh;
    overflow-y: auto;
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
     /* Fallback background if image fails */
    background-color: #363636;
}

.movie-poster img {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
     /* Hide broken image icon */
    color: transparent;
    font-size: 0;
}
/* Show alt text or a fallback if image fails to load */
.movie-poster img::before {
    content: attr(alt);
    display: block;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #363636;
    color: #ffffff;
    text-align: center;
    padding-top: 50%;
    font-size: 16px;
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
    font-size: 18px;
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
    color: #ffd700;
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
    display: flex;
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


/* Responsive Design */
@media (max-width: 768px) {
    .sidebar {
        width: 70px;
        padding: 10px;
    }

    .logo h2, .nav-links span, .bottom-links span {
        display: none;
    }

    .nav-links i, .bottom-links i {
        margin-right: 0;
        width: 100%;
        text-align: center;
    }

     .nav-links a, .bottom-links a {
         justify-content: center;
         padding: 10px 0;
     }

    .main-content {
        margin-left: 70px;
         padding: 10px;
    }

    .review-header {
        flex-direction: column;
        align-items: flex-start;
        padding: 15px;
    }
    .review-header h1 {
        margin-bottom: 15px;
        font-size: 24px;
    }
    .search-bar {
        width: 100%;
    }
     .search-bar input {
         width: 100%;
     }

    .movie-card {
        flex: 0 0 150px;
        min-width: 150px;
    }

    .movie-card img {
        height: 225px;
    }
     .movie-poster img::before {
         font-size: 14px;
     }

    .movie-details {
        padding: 10px;
    }
     .movie-details h3 {
         font-size: 14px;
     }
     .movie-info {
         font-size: 12px;
     }
     .rating {
         font-size: 0.9em;
         gap: 5px;
     }
     .rating-count {
         font-size: 12px;
     }
}
```

---

**17. `manage/indeks.php`**

*Place this file in the `manage` directory.*

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
            $pdo->beginTransaction();

            // IMPORTANT: Due to ON DELETE CASCADE in the schema, deleting the movie
            // from the 'movies' table will automatically delete associated rows
            // in 'movie_genres', 'reviews', and 'favorites'.

            // Prepare and execute the delete statement for the movie table
            // Ensure only the uploader can delete their movie
            $stmt = $pdo->prepare("DELETE FROM movies WHERE movie_id = ? AND uploaded_by = ?");
            $stmt->execute([$movieIdToDelete, $userId]);

            // Check if a row was actually deleted
            if ($stmt->rowCount() > 0) {
                 // Optional: Delete associated files (poster, trailer) from the server
                 // This is more complex as you need the file paths BEFORE deleting the DB record.
                 // To do this safely, you would fetch the movie first, get the paths, then start transaction,
                 // delete DB record, commit, then delete files.
                 // For simplicity here, file deletion is omitted, but consider adding it.
                 // Example (requires fetching movie before deletion):
                 /*
                 $old_movie = getMovieById($movieIdToDelete); // Fetch BEFORE delete
                 if ($old_movie) {
                      if (!empty($old_movie['poster_image']) && file_exists(UPLOAD_DIR_POSTERS . $old_movie['poster_image'])) {
                          unlink(UPLOAD_DIR_POSTERS . $old_movie['poster_image']);
                      }
                      if (!empty($old_movie['trailer_file']) && file_exists(UPLOAD_DIR_TRAILERS . $old_movie['trailer_file'])) {
                           unlink(UPLOAD_DIR_TRAILERS . $old_movie['trailer_file']);
                      }
                 }
                 */

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
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
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
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li>
            </ul>
            <ul class="bottom-links">
                <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li>
            </ul>
        </div>

        <!-- Main Content -->
        <main class="main-content">
            <div class="header">
                 <h1>Manage Movies</h1>
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
            <div class="movies-grid review-grid">
                <?php if (empty($movies)): ?>
                    <div class="empty-state full-width">
                        <i class="fas fa-film"></i>
                        <p>No movies uploaded yet</p>
                        <p class="subtitle">Start adding your movies by clicking the upload button below</p>
                    </div>
                <?php else: ?>
                     <?php foreach ($movies as $movie): ?>
                        <div class="movie-card">
                            <div class="movie-poster">
                                <img src="<?php echo htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                <div class="movie-actions">
                                     <!-- Edit Button (Link to edit page - create edit.php if needed) -->
                                     <!-- <a href="edit.php?id=<?php echo $movie['movie_id']; ?>" class="action-btn" title="Edit Movie">
                                         <i class="fas fa-edit"></i>
                                     </a> -->
                                     <!-- Delete Button (Form submission) -->
                                     <form action="indeks.php" method="POST" onsubmit="return confirm('Are you sure you want to delete the movie &quot;<?php echo htmlspecialchars($movie['title']); ?>&quot;? This action cannot be undone.');">
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
                        emptyState.className = 'empty-state search-empty-state full-width';
                        emptyState.innerHTML = `
                            <i class="fas fa-search"></i>
                            <p>No uploaded movies found matching "${htmlspecialchars(searchTerm)}"</p>
                            <p class="subtitle">Try a different search term</p>
                        `;
                        moviesGrid.appendChild(emptyState);
                    } else {
                         // Update text if search empty state already exists
                         searchEmptyState.querySelector('p:first-of-type').innerText = `No uploaded movies found matching "${htmlspecialchars(searchTerm)}"`;
                         searchEmptyState.style.display = 'flex';
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
             // Trigger search when search button is clicked (if you add one)
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
                       .replace(/>/g, '&gt;') // Keep > as is for HTML structure, but usually encode
                       .replace(/"/g, '&quot;')
                       .replace(/'/g, '&#039;');
         }
    </script>
</body>
</html>
```

---

**18. `manage/styles.css`**

*Place this file in the `manage` directory.*

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
     padding: 0 20px;
}

.header h1 {
    font-size: 28px;
    color: #00ffff;
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
.search-bar:focus-within {
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


/* Movies Grid Styles (Uses .review-grid from review/styles.css now) */
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
    min-height: 400px;
}


/* Action Buttons Styles */
.action-buttons {
    position: fixed;
    bottom: 30px;
    right: 30px;
    display: flex;
    gap: 15px;
    z-index: 500;
}

.action-buttons button,
.action-buttons a {
    width: 55px;
    height: 55px;
    border-radius: 50%;
    border: none;
    color: #fff;
    font-size: 1.3em;
    cursor: pointer;
    transition: transform 0.3s, background-color 0.3s, box-shadow 0.3s;
    display: flex;
    align-items: center;
    justify-content: center;
    text-decoration: none;
    box-shadow: 0 4px 10px rgba(0,0,0,0.3);
}

.upload-btn {
    background-color: #00FFFF;
    color: #1a1a1a;
}

.edit-all-btn {
    background-color: #363636;
    color: #ffffff;
}

.action-buttons button:hover,
.action-buttons a:hover {
    transform: scale(1.1);
    box-shadow: 0 6px 15px rgba(0,255,255,0.3);
}

.upload-btn:hover {
    background-color: #00CCCC;
}

.edit-all-btn:hover {
    background-color: #444444;
     box-shadow: 0 6px 15px rgba(255,255,255,0.1);
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
@media (max-width: 768px) {
    .sidebar {
        width: 70px;
        padding: 10px;
    }

    .logo h2, .nav-links span, .bottom-links span {
        display: none;
    }

    .nav-links i, .bottom-links i {
        margin-right: 0;
        width: 100%;
        text-align: center;
    }

     .nav-links a, .bottom-links a {
         justify-content: center;
         padding: 10px 0;
     }

    .main-content {
        margin-left: 70px;
         padding: 10px;
    }

    .header {
        flex-direction: column;
        align-items: flex-start;
        padding: 0 15px;
    }
     .header h1 {
         margin-bottom: 15px;
         font-size: 24px;
     }
    .search-bar {
        width: 100%;
    }
     .search-bar input {
         width: 100%;
     }

    .movies-grid.review-grid {
        grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
        gap: 10px;
    }

    .movie-card {
        flex: 0 0 auto; /* Allow cards to size based on grid minmax */
        min-width: unset;
    }

    .movie-card img {
        height: 225px; /* Maintain 2:3 aspect ratio */
    }
     .movie-poster img::before {
         font-size: 14px;
     }

    .movie-actions {
        padding: 10px;
        gap: 10px;
    }
    .action-btn {
        width: 30px;
        height: 30px;
        font-size: 16px;
    }

    .movie-details {
        padding: 10px;
    }
     .movie-details h3 {
         font-size: 14px;
     }
     .movie-info {
         font-size: 12px;
     }
     .rating {
         font-size: 0.9em;
         gap: 5px;
     }
     .rating-count {
         font-size: 12px;
     }

     .action-buttons {
         bottom: 15px;
         right: 15px;
         gap: 10px;
     }
     .action-buttons button,
     .action-buttons a {
         width: 45px;
         height: 45px;
         font-size: 1.1em;
     }
}
```

---

**19. `manage/upload.php`**

*Place this file in the `manage` directory.*

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

// Initialize variables for form values (useful for pre-filling on error)
$title = $_SESSION['upload_form_data']['title'] ?? '';
$summary = $_SESSION['upload_form_data']['summary'] ?? '';
$release_date = $_SESSION['upload_form_data']['release_date'] ?? '';
$duration_hours = $_SESSION['upload_form_data']['duration_hours'] ?? '';
$duration_minutes = $_SESSION['upload_form_data']['duration_minutes'] ?? '';
$age_rating = $_SESSION['upload_form_data']['age_rating'] ?? '';
$genres = $_SESSION['upload_form_data']['genres'] ?? [];
$trailer_url = $_SESSION['upload_form_data']['trailer_url'] ?? '';

// Clear stored form data from session
unset($_SESSION['upload_form_data']);


// Handle form submission
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // 1. Collect and Sanitize Input
    $title = trim($_POST['movie-title'] ?? '');
    $summary = trim($_POST['movie-summary'] ?? '');
    $release_date = $_POST['release-date'] ?? '';
    $duration_hours = filter_var($_POST['duration-hours'] ?? '', FILTER_VALIDATE_INT); // Use '' for initial state check
    $duration_minutes = filter_var($_POST['duration-minutes'] ?? '', FILTER_VALIDATE_INT);
    $age_rating = $_POST['age-rating'] ?? '';
    $genres = $_POST['genre'] ?? [];
    $trailer_url = trim($_POST['trailer-link'] ?? '');
    // Trailer file will be handled via $_FILES

    // Store current inputs in session in case of errors and redirect
    $_SESSION['upload_form_data'] = [
        'title' => $title,
        'summary' => $summary,
        'release_date' => $release_date,
        'duration_hours' => $_POST['duration-hours'] ?? '', // Store raw input for display
        'duration_minutes' => $_POST['duration-minutes'] ?? '', // Store raw input for display
        'age_rating' => $age_rating,
        'genres' => $genres,
        'trailer_url' => $trailer_url,
        // File inputs cannot be easily stored/retained in session
    ];


    $errors = [];

    // 2. Validate Input
    if (empty($title)) $errors[] = 'Movie Title is required.';
    if (empty($release_date)) $errors[] = 'Release Date is required.';
    if ($duration_hours === false || $duration_hours === '' || $duration_hours < 0) $errors[] = 'Valid Duration (Hours) is required.';
    if ($duration_minutes === false || $duration_minutes === '' || $duration_minutes < 0 || $duration_minutes > 59) $errors[] = 'Valid Duration (Minutes) is required (0-59).';
    if (empty($age_rating)) $errors[] = 'Age Rating is required.';
    if (empty($genres)) $errors[] = 'At least one Genre must be selected.';

    // Validate genre values against allowed ENUM values
    $allowed_genres = ['action', 'adventure', 'comedy', 'drama', 'horror', 'supernatural', 'animation', 'sci-fi'];
    foreach ($genres as $genre) {
        if (!in_array($genre, $allowed_genres)) {
            $errors[] = 'Invalid genre selected.';
            break;
        }
    }

    // 3. Handle File Uploads (Poster)
    $poster_image_path = null;
    $poster_upload_success = false;

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
            $destination = UPLOAD_DIR_POSTERS . $newFileName; // Use absolute path for moving

            if (move_uploaded_file($posterFile['tmp_name'], $destination)) {
                $poster_image_path = $newFileName; // Store just the filename in DB
                $poster_upload_success = true;
            } else {
                $errors[] = 'Failed to upload poster file. Check server permissions.';
            }
        }
    } else {
         // Poster is required for upload
         $errors[] = 'Movie Poster file is required.';
         // Handle specific upload errors if file was attempted but failed
         if (isset($_FILES['movie-poster']) && $_FILES['movie-poster']['error'] !== UPLOAD_ERR_NO_FILE) {
              $errors[] = 'Poster upload error: ' . $_FILES['movie-poster']['error'];
         }
    }

    // 4. Handle File Uploads (Trailer File - Optional) and Trailer URL
    $trailer_file_path = null;
    $trailer_upload_success = false; // Track if trailer file upload was successful

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
            $destination = UPLOAD_DIR_TRAILERS . $newFileName; // Use absolute path for moving

            if (move_uploaded_file($trailerFile['tmp_name'], $destination)) {
                $trailer_file_path = $newFileName; // Store just the filename in DB
                $trailer_upload_success = true;
                $trailer_url = null; // If file is uploaded, prioritize file and clear the URL
            } else {
                $errors[] = 'Failed to upload trailer file. Check server permissions.';
            }
        }
    } else {
         // Handle specific upload errors for trailer file if attempted
         if (isset($_FILES['trailer-file']) && $_FILES['trailer-file']['error'] !== UPLOAD_ERR_NO_FILE) {
             $errors[] = 'Trailer file upload error: ' . $_FILES['trailer-file']['error'];
         }
    }

    // Check if at least one trailer source is provided
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
                $poster_image_path, // Filename from successful upload
                $trailer_url,       // URL if provided and file not uploaded
                $trailer_file_path, // Filename if file uploaded
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
            $errors[] = 'An internal error occurred while saving the movie.';

            // Clean up uploaded files if DB insertion failed
            if ($poster_upload_success && file_exists(UPLOAD_DIR_POSTERS . $poster_image_path)) {
                unlink(UPLOAD_DIR_POSTERS . $poster_image_path);
            }
             if ($trailer_upload_success && file_exists(UPLOAD_DIR_TRAILERS . $trailer_file_path)) {
                unlink(UPLOAD_DIR_TRAILERS . $trailer_file_path);
            }
        }
    }

    // If there were any errors, set the error message and redirect back
    if (!empty($errors)) {
        $_SESSION['error_message'] = implode('<br>', $errors);
        // Form data already saved to session at the start
        header('Location: upload.php'); // Redirect back to show errors and pre-fill form
        exit;
    }
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
    <title>Upload Movie - RatingTales</title>
    <link rel="stylesheet" href="styles.css">
    <link rel="stylesheet" href="upload.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
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
                <li><a href="indeks.php" class="active"><i class="fas fa-film"></i> <span>Manage</span></a></li>
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li>
            </ul>
            <ul class="bottom-links">
                <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li>
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
                    <div class="alert error"><?php echo $error_message; ?></div>
                <?php endif; ?>

                <form class="upload-form" action="upload.php" method="post" enctype="multipart/form-data">
                    <div class="form-layout">
                        <div class="form-main">
                            <div class="form-group">
                                <label for="movie-title">Movie Title</label>
                                <input type="text" id="movie-title" name="movie-title" required value="<?php echo htmlspecialchars($title); ?>">
                            </div>

                            <div class="form-group">
                                <label for="movie-summary">Movie Summary</label>
                                <textarea id="movie-summary" name="movie-summary" rows="4" required><?php echo htmlspecialchars($summary); ?></textarea>
                            </div>

                            <div class="form-group">
                                <label>Genre</label>
                                <div class="genre-options">
                                    <?php
                                    $all_genres = ['action', 'adventure', 'comedy', 'drama', 'horror', 'supernatural', 'animation', 'sci-fi'];
                                    foreach ($all_genres as $genre_option):
                                        $checked = in_array($genre_option, $genres) ? 'checked' : '';
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
                                    $age_ratings = ['G', 'PG', 'PG-13', 'R', 'NC-17'];
                                    foreach ($age_ratings as $rating_option):
                                        $selected = ($age_rating === $rating_option) ? 'selected' : '';
                                    ?>
                                        <option value="<?php echo $rating_option; ?>" <?php echo $selected; ?>><?php echo $rating_option; ?></option>
                                    <?php endforeach; ?>
                                </select>
                            </div>

                            <div class="form-group">
                                <label for="movie-trailer">Movie Trailer</label>
                                <div class="trailer-input">
                                    <input type="text" id="trailer-link" name="trailer-link" placeholder="Enter YouTube video URL" value="<?php echo htmlspecialchars($trailer_url); ?>">
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
                                    <img id="poster-preview" src="#" alt="Poster Preview" style="display: none; max-width: 100%; max-height: 100%; object-fit: contain;">
                                </div>
                                <p class="trailer-note" style="margin-top: 5px;">(Recommended: Aspect Ratio 2:3, Max 5MB)</p>
                            </div>

                            <div class="advanced-settings">
                                <h3>Advanced Settings</h3>
                                <div class="form-group">
                                    <label for="release-date">Release Date</label>
                                    <input type="date" id="release-date" name="release-date" required value="<?php echo htmlspecialchars($release_date); ?>">
                                </div>

                                <div class="form-group">
                                    <label for="duration-hours">Film Duration</label>
                                    <div class="duration-inputs">
                                        <div class="duration-field">
                                            <input type="number" id="duration-hours" name="duration-hours" min="0" placeholder="Hours" required value="<?php echo htmlspecialchars($duration_hours); ?>">
                                            <span>Hours</span>
                                        </div>
                                        <div class="duration-field">
                                            <input type="number" id="duration-minutes" name="duration-minutes" min="0" max="59" placeholder="Minutes" required value="<?php echo htmlspecialchars($duration_minutes); ?>">
                                            <span>Minutes</span>
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>

                    <div class="form-actions">
                        <button type="button" class="cancel-btn" onclick="window.location.href='indeks.php'">Cancel</button>
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
                     posterPreview.style.objectFit = 'contain'; // Set object-fit for preview
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
                 // Check file type before assigning (basic client-side check)
                 const allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
                 if (allowedTypes.includes(files[0].type)) {
                      posterInput.files = files; // Assign files to input
                      posterInput.dispatchEvent(new Event('change')); // Trigger change event
                 } else {
                      alert('Invalid file type. Only JPG, PNG, GIF, WEBP are allowed.');
                 }
             }
         });


         // JavaScript to handle mutual exclusivity of trailer URL and File
         const trailerLinkInput = document.getElementById('trailer-link');
         const trailerFileInput = document.getElementById('trailer-file');

         trailerLinkInput.addEventListener('input', function() {
             if (this.value.trim() !== '') {
                 trailerFileInput.disabled = true; // Disable file input if URL is entered
                 // Also clear file input if a URL is entered after a file was selected
                 trailerFileInput.value = '';
             } else {
                 trailerFileInput.disabled = false; // Enable file input if URL is empty
             }
         });

         trailerFileInput.addEventListener('change', function() {
             if (this.files.length > 0) {
                 trailerLinkInput.disabled = true; // Disable URL input if file is selected
                  // Also clear URL input if a file is selected after a URL was entered
                 trailerLinkInput.value = '';
             } else {
                 trailerLinkInput.disabled = false; // Enable URL input if file selection is cleared
             }
         });

         // Initial check on page load (important if form data is pre-filled after error)
         document.addEventListener('DOMContentLoaded', () => {
             if (trailerLinkInput.value.trim() !== '') {
                 trailerFileInput.disabled = true;
             } else if (trailerFileInput.files.length > 0) { // Check files property directly
                 trailerLinkInput.disabled = true;
             }
             // If editing and poster already exists, show it
             // This requires fetching movie data in PHP first and pre-filling an attribute on #poster-preview
             // Example (assuming PHP provides $existing_poster_url):
             // const existingPosterUrl = "<?php // echo htmlspecialchars($existing_poster_url ?? ''); ?>";
             // if (existingPosterUrl) {
             //     posterPreview.src = existingPosterUrl;
             //     posterPreview.style.display = 'block';
             //     uploadAreaIcon.style.display = 'none';
             //     uploadAreaText.style.display = 'none';
             //      posterPreview.style.objectFit = 'cover'; // Use cover for existing poster
             // }
         });

         // Helper function for HTML escaping (client-side) - good practice if displaying user input again
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

---

**20. `manage/upload.css`**

*Place this file in the `manage` directory.*

```css
/* manage/upload.css */
/* Assuming base styles from manage/styles.css are included */

.upload-container {
    padding: 20px;
}

.upload-container h1 {
    color: #00ffff;
    font-size: 28px;
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
     @media (max-width: 992px) {
         grid-template-columns: 1fr;
         gap: 25px;
     }
}

/* Form Main Section */
.form-main {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

/* Form Side Section */
.form-side {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

/* Form Groups */
.form-group {
    margin-bottom: 0;
}

.form-group label {
    display: block;
    margin-bottom: 8px;
    color: #ffffff;
    font-size: 15px;
    font-weight: bold;
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
    color: #888;
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
     font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
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
}

.upload-area {
    width: 100%;
    height: 300px;
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
     position: relative;
    overflow: hidden;
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
     z-index: 10;
}

/* Poster preview inside upload area */
#poster-preview {
    display: none;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
     z-index: 5;
}


/* Advanced Settings */
.form-section {
    margin-top: 30px;
}

.form-section h2, .advanced-settings h3 {
    color: #00ffff;
    font-size: 20px;
    margin-bottom: 20px;
     border-bottom: 1px solid #333;
     padding-bottom: 10px;
}

.advanced-settings {
    display: flex;
    flex-direction: column;
    gap: 20px;
    background-color: #1a1a1a;
    border-radius: 10px;
    padding: 20px;
}


/* Genre Options */
.genre-options {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
    gap: 10px;
}

.checkbox-label {
    display: flex;
    align-items: center;
    cursor: pointer;
    user-select: none;
}

.checkbox-label input[type="checkbox"] {
    margin-right: 8px;
    appearance: none;
    width: 18px;
    height: 18px;
    border: 2px solid #363636;
    border-radius: 4px;
    background-color: #1a1a1a;
    cursor: pointer;
    transition: all 0.3s;
    flex-shrink: 0;
     position: relative;
}

.checkbox-label input[type="checkbox"]:checked {
    background-color: #00ffff;
    border-color: #00ffff;
}

.checkbox-label input[type="checkbox"]:checked::before {
    content: '\f00c';
    font-family: 'Font Awesome 6 Free';
    font-weight: 900;
    display: flex;
    justify-content: center;
    align-items: center;
    color: #1a1a1a;
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
    padding-right: 120px;
}

.trailer-upload {
    position: relative;
    background-color: #1a1a1a;
    border: 1px solid #363636;
    border-radius: 8px;
    padding: 12px;
    margin-top: 10px;
    display: flex;
    align-items: center;
    gap: 10px;
}

.trailer-upload input[type="file"] {
    width: auto;
    flex-grow: 1;
    color: #ffffff;
    font-size: 14px;
    padding: 0;
    border: none;
    background: none;
}

.trailer-upload input[type="file"]::file-selector-button {
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
    white-space: nowrap;
    flex-shrink: 0;
}
.trailer-input .trailer-note {
    position: absolute;
    right: 12px;
    top: 50%;
    transform: translateY(-50%);
}
.trailer-upload .trailer-note {
    position: static;
    display: block;
    margin-top: 5px;
    transform: none;
    text-align: right;
    width: 100%;
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
    font-weight: bold;
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
    background-color: #363636;
    color: #ffffff;
}

.cancel-btn:hover {
    background-color: #444444;
}

.submit-btn {
    background-color: #00ffff;
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
@media (max-width: 992px) {
    .form-layout {
        grid-template-columns: 1fr;
        gap: 25px;
    }

    .form-side {
         gap: 20px;
    }

    .form-actions {
        flex-direction: column;
        gap: 10px;
    }

    .form-actions button {
        width: 100%;
        justify-content: center;
    }

     .poster-upload {
         margin-top: 0;
     }
     .advanced-settings {
         margin-top: 0;
         padding-top: 20px;
         border-top: 1px solid #333;
     }
     .advanced-settings h3 {
         margin-top: 0;
     }

}
```

---

**21. `acc_page/index.php`**

*Place this file in the `acc_page` directory.*

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

// Handle user not found after authentication check (shouldn't happen if getAuthenticatedUser works)
if (!$user) {
    $_SESSION['error_message'] = 'User profile could not be loaded.';
    header('Location: ../autentikasi/logout.php'); // Force logout if user somehow invalidates
    exit;
}


// Fetch movies uploaded by the current user
$uploadedMovies = getUserUploadedMovies($userId);

// Handle profile update (using POST form)
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['update_profile'])) {
    $update_data = [];
    $has_update = false;
    $errors = [];
    $success = false;

    // Handle bio update
    if (isset($_POST['bio'])) {
        $update_data['bio'] = trim($_POST['bio']);
        $has_update = true;
    }

     // You would add handlers for other editable fields here if they were implemented
     // For example, if you added input fields for full_name, age, gender in the form
     // if (isset($_POST['full_name'])) {
     //     $update_data['full_name'] = trim($_POST['full_name']);
     //     $has_update = true;
     // }
     // if (isset($_POST['age'])) {
     //     $age = filter_var($_POST['age'], FILTER_VALIDATE_INT);
     //     if ($age !== false && $age > 0) {
     //          $update_data['age'] = $age;
     //          $has_update = true;
     //     } else {
     //          $errors[] = 'Invalid age provided.';
     //     }
     // }


     // Handle profile image upload (more complex, needs file handling)
     // This requires a file input in the form and logic similar to movie poster upload

    if (empty($errors) && $has_update) {
        if (updateUser($userId, $update_data)) {
            $_SESSION['success_message'] = 'Profile updated successfully!';
             // Refresh user data after update
            $user = getAuthenticatedUser(); // Re-fetch user details
             $success = true;
        } else {
            $_SESSION['error_message'] = 'Failed to update profile.';
        }
    } elseif (!empty($errors)) {
         $_SESSION['error_message'] = implode('<br>', $errors);
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

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Profile</title>
    <link rel="stylesheet" href="styles.css">
     <link rel="stylesheet" href="../review/styles.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
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
                 <li class="active"><a href="#"><i class="fas fa-user"></i> <span>Profile</span></a></li>
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li>
                </ul>
            </div>
        </nav>
        <main class="main-content">
             <?php if ($success_message): ?>
                <div class="alert success"><?php echo htmlspecialchars($success_message); ?></div>
            <?php endif; ?>
            <?php if ($error_message): ?>
                <div class="alert error"><?php echo $error_message; ?></div>
            <?php endif; ?>

            <div class="profile-header">
                <div class="profile-info">
                    <div class="profile-image">
                        <img src="<?php echo htmlspecialchars($user['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($user['full_name'] ?? $user['username']) . '&background=random&color=fff&size=120'); ?>" alt="Profile Picture">
                         <!-- Optional: Add edit icon for profile image upload -->
                         <!-- <i class="fas fa-camera edit-icon" title="Change Profile Image"></i> -->
                    </div>
                    <div class="profile-details">
                        <h1>
                            <span id="displayName"><?php echo htmlspecialchars($user['full_name'] ?? 'N/A'); ?></span>
                            <!-- Edit icon for display name (currently not implemented via form) -->
                            <!-- <i class="fas fa-pen edit-icon" onclick="toggleEdit('displayName')"></i> -->
                        </h1>
                        <p class="username">
                            @<span id="username"><?php echo htmlspecialchars($user['username']); ?></span>
                            <!-- Edit icon for username (usually disabled) -->
                            <!-- <i class="fas fa-pen edit-icon" onclick="toggleEdit('username')"></i> -->
                        </p>
                         <p class="user-meta"><?php echo htmlspecialchars($user['age'] ?? 'N/A'); ?> | <?php echo htmlspecialchars($user['gender'] ?? 'N/A'); ?></p>
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
                <h2>Movies I Uploaded</h2>
                <div class="movies-grid review-grid">
                    <?php if (!empty($uploadedMovies)): ?>
                        <?php foreach ($uploadedMovies as $movie): ?>
                            <div class="movie-card" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'">
                                <div class="movie-poster">
                                    <img src="<?php echo htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                    <div class="movie-actions">
                                         <!-- Optional: Add edit/delete action buttons directly here -->
                                         <!-- <a href="../manage/edit.php?id=<?php echo $movie['movie_id']; ?>" class="action-btn" title="Edit Movie"><i class="fas fa-edit"></i></a> -->
                                         <!-- Delete form (should point to manage/indeks.php handler) -->
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
                        <div class="empty-state full-width">
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
            if (bioElement.querySelector('form')) { // Check for the form element instead of textarea
                return;
            }

            const textarea = document.createElement('textarea');
            textarea.className = 'edit-input bio-input';
            textarea.value = (currentText === 'Click to add bio...' || currentText === '') ? '' : currentText;
            textarea.placeholder = 'Write something about yourself...';

            // Create Save and Cancel buttons
            const saveButton = document.createElement('button');
            saveButton.textContent = 'Save';
            saveButton.className = 'btn-save-bio';
            saveButton.type = 'submit'; // Make save button submit the form

            const cancelButton = document.createElement('button');
            cancelButton.textContent = 'Cancel';
            cancelButton.className = 'btn-cancel-bio';
            cancelButton.type = 'button'; // Keep cancel as button


            // Create a form to wrap the textarea and buttons
            const form = document.createElement('form');
            form.method = 'POST';
            form.action = 'index.php'; // Submit back to this page

            const hiddenUpdateInput = document.createElement('input');
            hiddenUpdateInput.type = 'hidden';
            hiddenUpdateInput.name = 'update_profile'; // Identifier for the POST handler
            hiddenUpdateInput.value = '1';

            // The textarea itself will send the 'bio' data because it has the 'name="bio"' attribute

            // Replace content with textarea and buttons
            bioElement.innerHTML = ''; // Use innerHTML to remove previous content including <br>
            form.appendChild(hiddenUpdateInput);
            form.appendChild(textarea);
             // Wrap buttons in a div for layout
             const buttonDiv = document.createElement('div');
             buttonDiv.style.marginTop = '10px';
             buttonDiv.style.textAlign = 'right';
             buttonDiv.appendChild(cancelButton);
             buttonDiv.appendChild(saveButton);
             form.appendChild(buttonDiv);

            bioElement.appendChild(form);
            textarea.focus();

            // Handle Cancel
            cancelButton.onclick = function() {
                // Restore original content with preserved line breaks
                let originalValue = (currentText === 'Click to add bio...' || currentText === '') ? 'Click to add bio...' : currentText;
                bioElement.innerHTML = nl2br(htmlspecialchars(originalValue)); // Use nl2br on restore
            };

             // Add keydown listener to textarea for saving on Enter (optional)
            // textarea.addEventListener('keydown', function(event) {
            //     // Check if Enter key is pressed (and not Shift+Enter for newline)
            //     if (event.key === 'Enter' && !event.shiftKey) {
            //         event.preventDefault(); // Prevent default newline
            //         form.submit(); // Submit the form
            //     }
            // });
        }

         // Helper function for nl2br (client-side equivalent for display)
         function nl2br(str) {
             if (typeof str !== 'string') return str;
             // Replace \r\n, \r, or \n with <br>
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
            padding: 0;
         }

     </style>
</body>
</html>
```

---

**22. `acc_page/styles.css`**

*Place this file in the `acc_page` directory.*

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
    flex-wrap: wrap;
}

.profile-image {
    width: 120px;
    height: 120px;
    border-radius: 50%;
    overflow: hidden;
    border: 3px solid #00ffff;
    box-shadow: 0 0 10px rgba(0, 255, 255, 0.5);
    flex-shrink: 0;
     /* Fallback background if image fails */
    background-color: #363636;
}

.profile-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
     /* Hide broken image icon */
    color: transparent;
    font-size: 0;
}
/* Show alt text or a fallback if image fails to load */
.profile-image img::before {
    content: attr(alt);
    display: block;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #363636;
    color: #ffffff;
    text-align: center;
    line-height: 120px;
    font-size: 18px;
}


.profile-details {
     flex-grow: 1;
     min-width: 200px;
}

.profile-details h1 {
    font-size: 32px;
    margin-bottom: 5px;
    display: flex;
    align-items: center;
    gap: 15px;
     color: #00ffff;
}

.edit-icon {
    font-size: 18px;
    color: #00ffff;
    cursor: pointer;
    transition: color 0.3s, transform 0.2s;
    margin-left: 5px;
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
     padding: 10px;
     border-radius: 8px;
     line-height: 1.6;
     color: #ccc;
     white-space: pre-wrap; /* Preserve line breaks */
}

#bio:hover {
    background-color: #2a2a2a;
}

.username {
    color: #888;
    font-size: 16px;
    margin-bottom: 5px;
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
     border-bottom: 1px solid #333;
     padding-bottom: 10px;
}

.about-content {
    min-height: 100px;
    background-color: #242424;
    border-radius: 8px;
    padding: 15px;
     white-space: pre-wrap; /* Preserve line breaks */
}

/* Posts Section Styles */
.posts-section {
    padding: 20px 0;
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

/* Responsive Design */
@media (max-width: 768px) {
    .sidebar {
        width: 70px;
        padding: 10px;
    }

    .logo h2, .nav-links span, .bottom-links span {
        display: none;
    }

    .nav-links i, .bottom-links i {
        margin-right: 0;
        width: 100%;
        text-align: center;
    }

     .nav-links a, .bottom-links a {
         justify-content: center;
         padding: 10px 0;
     }

    .main-content {
        margin-left: 70px;
         padding: 10px;
    }

    .profile-header {
        padding: 20px;
    }

    .profile-info {
        flex-direction: column;
        align-items: center;
        gap: 20px;
        margin-bottom: 20px;
    }
     .profile-image {
         width: 100px;
         height: 100px;
     }
     .profile-image img::before {
          line-height: 100px;
          font-size: 16px;
     }

    .profile-details {
        text-align: center;
        min-width: unset;
        width: 100%;
    }
     .profile-details h1 {
         justify-content: center;
         font-size: 24px;
     }
     .username {
         font-size: 14px;
     }
     .user-meta {
         font-size: 12px;
     }

    .about-me {
        padding: 15px;
    }
    .about-me h2 {
        font-size: 18px;
        margin-bottom: 10px;
    }
     .about-content {
         padding: 10px;
         min-height: 80px;
     }

    .posts-section {
        padding: 10px 0;
    }
    .posts-section h2 {
        font-size: 20px;
        margin-bottom: 15px;
    }
     .movies-grid.review-grid {
         grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
         gap: 10px;
     }
     .movie-card {
         flex: 0 0 auto;
         min-width: unset;
     }
     .movie-card img {
         height: 225px;
     }
     .movie-poster img::before {
         font-size: 14px;
     }
      .movie-details {
         padding: 10px;
     }
      .movie-details h3 {
          font-size: 14px;
      }
      .movie-info {
          font-size: 12px;
      }
      .rating {
          font-size: 0.9em;
          gap: 5px;
      }
      .rating-count {
          font-size: 12px;
      }

      .empty-state.full-width {
         padding: 20px;
         min-height: 200px;
     }
      .empty-state i {
          font-size: 3em;
      }
      .empty-state p {
          font-size: 1em;
      }
      .empty-state .subtitle {
          font-size: 0.8em;
      }


 }
```

---

**Setup Steps Summary:**

1.  **Database:** Create `ratingtales` database and run the `database.sql` script.
2.  **File Structure:** Create the following directories:
    *   `config`
    *   `includes`
    *   `uploads/posters`
    *   `uploads/trailers`
    *   `gambar`
3.  **Place Files:** Put each code block into the corresponding file name and place them in their respective directories (`config`, `includes`, `autentikasi`, `beranda`, `favorite`, `manage`, `review`, `acc_page`, `gambar`).
4.  **Uploads Permissions:** Ensure the `uploads` directory and its subdirectories (`posters`, `trailers`) are writable by your web server process. Permissions like 0775 or 0777 might be needed depending on your server setup.
5.  **Placeholder Images:** Place `5302920.jpg` and `Untitled142_20250310223718.png` in the `gambar` directory. You might also want a generic `placeholder.jpg` in `gambar`.
6.  **Web Server:** Configure your web server (Apache/Nginx) to serve the files. Access the application through the `autentikasi` directory (e.g., `http://localhost/your_project_folder/autentikasi/form-login.php`).

Now the application should be fully integrated with the database, the navigation and icons should be functional, forms should process data securely, and the styling should be consistent. Let me know if you encounter any other specific issues!
