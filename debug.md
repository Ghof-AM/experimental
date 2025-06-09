Okay, let's tackle this project step by step.

Based on the files you provided and the issues you're facing, here's a plan to fix the problems and integrate Google Sign-In:

1.  **Database Schema Update:** The `reviews` table needs a unique key for the `ON DUPLICATE KEY UPDATE` clause in `createReview` to work correctly (one review per user per movie). We also need to add a column for Google User IDs.
2.  **Code Refactoring:** Streamline helper functions and database interactions.
3.  **Fix Existing Issues:**
    *   Login/Logout: Review session management and redirects.
    *   Trailer Playback: Correct JavaScript logic for playing different trailer types (URL vs. File).
    *   Edit Movie: Create the missing `manage/edit.php` and the necessary database function.
    *   Bio Edit: Review the existing logic, which seems mostly correct, and ensure it works after other fixes.
4.  **Implement Google Sign-In:** Add the client-side integration and server-side verification logic.
5.  **Provide Instructions:** Detail how to set up the database changes, get Google API keys, and configure the project.

---

### Step 1: Database Schema Update

We need to add a `UNIQUE KEY` to the `reviews` table and add a `google_id` column to the `users` table.

Execute these SQL commands on your `ratingtales` database:

```sql
-- Add google_id to users table
ALTER TABLE users ADD COLUMN google_id VARCHAR(255) UNIQUE NULL AFTER email;

-- Add unique key to reviews table to allow ON DUPLICATE KEY UPDATE
-- Check if index exists before adding (prevents errors on re-running)
DO @exist := (SELECT COUNT(*) FROM INFORMATION_SCHEMA.STATISTICS WHERE table_schema = DATABASE() AND table_name = 'reviews' AND index_name = 'unique_user_movie_review');
SET @sql := IF (@exist > 0, 'SELECT ''Index already exists''', 'ALTER TABLE reviews ADD UNIQUE KEY unique_user_movie_review (movie_id, user_id)');
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;

-- Optional: You might want to add indexes on google_id for faster lookups
-- CREATE INDEX idx_users_google_id ON users(google_id);
```

---

### Step 2 & 3 & 5: Code Reconstruction and Fixes

I'll reconstruct the relevant PHP and HTML files. The CSS files seem mostly okay but I'll include them for completeness.

**Important Notes:**

*   **File Paths:** I'll generally use relative paths (`../`) which work well when accessing files from a standard web server structure (like `/R-TALES_EX-C1/`). If your setup is different (e.g., accessing directly from the root), you might need to adjust paths.
*   **Error Handling:** I've kept the session-based error/success messages (`$_SESSION['error_message']`, `$_SESSION['success_message']`) which is a simple pattern for redirects.
*   **Google Sign-In:** This requires a new file (`autentikasi/verify_google_login.php`) and adding the Google Client ID to your `config.php`. Instructions will follow the code.
*   **Manage Edit:** I've created the `manage/edit.php` file and the corresponding `updateMovie` function in `database.php`.

Here are the revised files:

---

**`includes/config.php`**

```php
<?php
// includes/config.php

// Start a session if one hasn't been started already
if (session_status() == PHP_SESSION_NONE) {
    session_start();
}

// Set timezone (optional but recommended)
date_default_timezone_set('Asia/Jakarta'); // Example: Jakarta timezone

// Define site root path (helps with absolute paths if needed, currently using relative)
// define('SITE_ROOT', '/R-TALES_EX-C1'); // Adjust if your project is not in a subfolder

// Include the database connection and functions file
require_once __DIR__ . '/../config/database.php'; // Use __DIR__ for reliable path

// Configure error reporting for debugging (Disable on production)
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

// Set up a basic error log (make sure the 'logs' directory exists and is writable)
// ini_set('log_errors', 1);
// ini_set('error_log', __DIR__ . '/../logs/php-error.log');

// --- Google Sign-In Configuration ---
// You need to get your Google Client ID from Google Cloud Console
// Instructions: https://developers.google.com/identity/gsi/web/guides/get-started
define('GOOGLE_CLIENT_ID', 'YOUR_GOOGLE_CLIENT_ID'); // <-- REPLACE THIS WITH YOUR ACTUAL CLIENT ID

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
            // Also unset Google login session marker if it exists
            unset($_SESSION['user_id']);
            unset($_SESSION['google_login']);
            session_regenerate_id(true); // Regenerate ID after clearing session
            header('Location: ../autentikasi/form-login.php'); // Redirect to login
            exit;
        }
    }
    return null; // Return null if not authenticated
}

?>
```

---

**`config/database.php`**

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

// --- User Functions ---

// Added full_name, age, gender, bio, google_id to createUser
function createUser($full_name, $username, $email, $password = null, $age = null, $gender = null, $google_id = null, $profile_image = null, $bio = null) {
    global $pdo;
    $hashedPassword = $password ? password_hash($password, PASSWORD_BCRYPT) : null; // Hash password if provided

    $stmt = $pdo->prepare("INSERT INTO users (full_name, username, email, password, age, gender, google_id, profile_image, bio) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)");
    return $stmt->execute([$full_name, $username, $email, $hashedPassword, $age, $gender, $google_id, $profile_image, $bio]);
}

// Get user by ID
function getUserById($userId) {
    global $pdo;
    $stmt = $pdo->prepare("SELECT user_id, full_name, username, email, profile_image, age, gender, bio FROM users WHERE user_id = ?");
    $stmt->execute([$userId]);
    return $stmt->fetch();
}

// Get user by Email
function getUserByEmail($email) {
     global $pdo;
     $stmt = $pdo->prepare("SELECT user_id, full_name, username, email, profile_image, age, gender, bio FROM users WHERE email = ?");
     $stmt->execute([$email]);
     return $stmt->fetch();
}

// Get user by Google ID
function getUserByGoogleId($googleId) {
     global $pdo;
     $stmt = $pdo->prepare("SELECT user_id, full_name, username, email, profile_image, age, gender, bio FROM users WHERE google_id = ?");
     $stmt->execute([$googleId]);
     return $stmt->fetch();
}


// Check if username or email already exists
function isUsernameOrEmailExists($username, $email, $excludeUserId = null) {
    global $pdo;

    $query = "SELECT COUNT(*) FROM users WHERE username = ? OR email = ?";
    $params = [$username, $email];

    if ($excludeUserId !== null) {
        $query .= " AND user_id != ?";
        $params[] = $excludeUserId;
    }

    $stmt = $pdo->prepare($query);
    $stmt->execute($params);
    return $stmt->fetchColumn() > 0;
}

// Added updateUser function - Handles multiple fields dynamically
function updateUser($userId, $data) {
    global $pdo;
    $updates = [];
    $params = [];
    $allowed_fields = ['full_name', 'username', 'email', 'profile_image', 'age', 'gender', 'bio', 'password']; // Added password

    foreach ($data as $key => $value) {
        // Basic validation for allowed update fields
        if (in_array($key, $allowed_fields)) {
            // Handle password hashing if included in data
            if ($key === 'password' && !empty($value)) {
                $value = password_hash($value, PASSWORD_BCRYPT);
            } else if ($key === 'password' && empty($value)) {
                 // Skip if password is empty (don't update password with empty value)
                 continue;
            }
             // Use backticks for column names in case they are reserved words
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


// --- Movie Functions ---

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

// Function to remove all genres for a movie
function removeMovieGenres($movie_id) {
    global $pdo;
    $stmt = $pdo->prepare("DELETE FROM movie_genres WHERE movie_id = ?");
    return $stmt->execute([$movie_id]);
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
        // Ensure genres is an array if needed, currently a comma-separated string
         if (!empty($movie['genres'])) {
             $movie['genres_array'] = explode(', ', $movie['genres']);
         } else {
             $movie['genres_array'] = [];
         }
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
    $movies = $stmt->fetchAll();

    // Convert genres string to array for consistency
    foreach ($movies as &$movie) {
         if (!empty($movie['genres'])) {
             $movie['genres_array'] = explode(', ', $movie['genres']);
         } else {
             $movie['genres_array'] = [];
         }
    }
     unset($movie); // Break the reference with the last element

    return $movies;
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
     $movies = $stmt->fetchAll();

     // Convert genres string to array for consistency
     foreach ($movies as &$movie) {
          if (!empty($movie['genres'])) {
              $movie['genres_array'] = explode(', ', $movie['genres']);
          } else {
              $movie['genres_array'] = [];
          }
     }
      unset($movie); // Break the reference with the last element

     return $movies;
}

// Added function to update a movie
function updateMovie($movieId, $data) {
    global $pdo;
    $updates = [];
    $params = [];
    $allowed_fields = ['title', 'summary', 'release_date', 'duration_hours', 'duration_minutes', 'age_rating', 'poster_image', 'trailer_url', 'trailer_file'];

    foreach ($data as $key => $value) {
        if (in_array($key, $allowed_fields)) {
            $updates[] = "`{$key}` = ?";
            $params[] = $value;
        }
    }

    if (empty($updates)) {
        return false; // Nothing to update
    }

    $sql = "UPDATE movies SET " . implode(', ', $updates) . " WHERE movie_id = ?";
    $params[] = $movieId;

    $stmt = $pdo->prepare($sql);
    return $stmt->execute($params);
}


// --- Review Functions ---

// Supports inserting a new review or updating an existing one (upsert)
// Relies on the UNIQUE KEY unique_user_movie_review (movie_id, user_id) added to the reviews table
function createReview($movie_id, $user_id, $rating, $comment) {
    global $pdo;
    // Using ON DUPLICATE KEY UPDATE now that the unique key exists
    $stmt = $pdo->prepare("
        INSERT INTO reviews (movie_id, user_id, rating, comment)
        VALUES (?, ?, ?, ?)
        ON DUPLICATE KEY UPDATE rating = VALUES(rating), comment = VALUES(comment), created_at = CURRENT_TIMESTAMP -- Update timestamp on update
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
            u.full_name, -- Fetch full_name too, might be useful
            u.profile_image
        FROM reviews r
        JOIN users u ON r.user_id = u.user_id
        WHERE r.movie_id = ?
        ORDER BY r.created_at DESC
    ");
    $stmt->execute([$movieId]);
    return $stmt->fetchAll();
}

// Function to get a single user's review for a specific movie
function getUserReviewForMovie($movieId, $userId) {
     global $pdo;
     $stmt = $pdo->prepare("
         SELECT
             r.*,
             u.username,
             u.full_name,
             u.profile_image
         FROM reviews r
         JOIN users u ON r.user_id = u.user_id
         WHERE r.movie_id = ? AND r.user_id = ? LIMIT 1
     ");
     $stmt->execute([$movieId, $userId]);
     return $stmt->fetch();
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


// --- Favorite Functions ---

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
    $movies = $stmt->fetchAll();

     // Convert genres string to array for consistency
     foreach ($movies as &$movie) {
          if (!empty($movie['genres'])) {
              $movie['genres_array'] = explode(', ', $movie['genres']);
          } else {
              $movie['genres_array'] = [];
          }
     }
      unset($movie); // Break the reference with the last element

     return $movies;
}

// Helper function to check if a movie is favorited by the current user
function isMovieFavorited($movieId, $userId) {
    global $pdo;
    if (!$userId) return false; // Cannot favorite if not logged in
    $stmt = $pdo->prepare("SELECT COUNT(*) FROM favorites WHERE movie_id = ? AND user_id = ?");
    $stmt->execute([$movieId, $userId]);
    return $stmt->fetchColumn() > 0;
}


?>
```

---

**`autentikasi/animation.js`** (No change needed)

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

**`autentikasi/form-login.php`**

```php
<?php
// autentikasi/form-login.php
require_once '../includes/config.php'; // Include config.php and database functions

// Redirect if already authenticated
if (isAuthenticated()) {
    header('Location: ../beranda/index.php');
    exit;
}

// --- Logika generate CAPTCHA (Server-side) ---
// Generate CAPTCHA new if not set or if redirected back due to error
if (!isset($_SESSION['captcha_code']) || isset($_SESSION['error_message'])) {
     $_SESSION['captcha_code'] = generateRandomString(6);
}

// --- Proses Form Login ---
$error_message = null;
$success_message = null;

// Retrieve input values from session if redirected back due to error (improves UX)
$username_input = $_SESSION['login_username_input'] ?? '';
// CAPTCHA input is intentionally NOT pre-filled for security

// Get messages from session and clear them
if (isset($_SESSION['error_message'])) {
    $error_message = $_SESSION['error_message'];
    unset($_SESSION['error_message']);
}
if (isset($_SESSION['success_message'])) {
    $success_message = $_SESSION['success_message'];
    unset($_SESSION['success_message']);
}

// Clear stored inputs from session AFTER retrieving messages
unset($_SESSION['login_username_input']);


if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // Ambil dan bersihkan input
    $username_input_post = trim($_POST['username'] ?? '');
    $password_input = $_POST['password'] ?? '';
    $captcha_input_post = trim($_POST['captcha_input'] ?? '');

    // Store inputs in session in case of redirect (for UX)
    $_SESSION['login_username_input'] = $username_input_post;

    // --- Validasi Server-side ---
    $errors = [];
    if (empty($username_input_post)) $errors[] = 'Username/Email is required.';
    if (empty($password_input)) $errors[] = 'Password is required.';
    if (empty($captcha_input_post)) $errors[] = 'CAPTCHA is required.';

    // --- Validasi CAPTCHA ---
    // Check CAPTCHA only if other required fields are present to avoid regenerating CAPTCHA on empty fields error
    if (empty($errors)) {
         if (!isset($_SESSION['captcha_code']) || strtolower($captcha_input_post) !== strtolower($_SESSION['captcha_code'])) {
             $errors[] = 'Invalid CAPTCHA.';
             // CAPTCHA is regenerated at the top of the file if there are errors
         } else {
             // CAPTCHA valid, unset it immediately to prevent reuse
             unset($_SESSION['captcha_code']);
         }
    }


    // If there are validation errors, store them in session and redirect
    if (!empty($errors)) {
        $_SESSION['error_message'] = implode('<br>', $errors);
        header('Location: form-login.php'); // Redirect back to show error and new CAPTCHA
        exit;
    }

    // --- If all validations pass, proceed to DB checks ---
    try {
        // Cek pengguna berdasarkan username atau email
        // Using getUserByEmail from database.php which handles both cases via a single query
        $user = getUserByEmail($username_input_post);

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
            $_SESSION['error_message'] = 'Incorrect Username/Email or password.';
             // CAPTCHA is regenerated at the top of the file if there are errors
            header('Location: form-login.php');
            exit;
        }

    } catch (PDOException $e) {
        error_log("Database error during login: " . $e->getMessage());
        $_SESSION['error_message'] = 'An internal error occurred. Please try again.';
         // CAPTCHA is regenerated at the top of the file if there are errors
        header('Location: form-login.php');
        exit;
    }
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
    <!-- Google Sign-In -->
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
            </div>

            <button type="submit" class="btn">Login</button>
        </form>

        <div class="form-link separator">OR</div>

        <!-- Google Sign-In Button -->
        <div class="google-signin-container">
            <!-- The following div is provided by Google Identity Services -->
            <div id="g_id_onload"
                 data-client_id="<?php echo GOOGLE_CLIENT_ID; ?>"
                 data-callback="handleCredentialResponse"
                 data-auto_prompt="false">
            </div>
            <div class="g_id_signin"
                 data-type="standard"
                 data-size="large"
                 data-theme="filled_blue"
                 data-text="signin_with"
                 data-shape="rectangular"
                 data-logo_alignment="left">
            </div>
        </div>


        <p class="form-link">Don't have an account? <a href="form-register.php">Register here</a></p>
    </div>

    <script src="animation.js"></script>
    <script>
        // Variable to store the current CAPTCHA code
        // Using PHP to insert the code from the session
        let currentCaptchaCode = "<?php echo htmlspecialchars($captchaCodeForClient); ?>";

        const captchaInput = document.getElementById('captchaInput');
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
            } catch (error) {
                console.error("Error generating CAPTCHA:", error);
                // Optionally display an error message on the page
            }
        }

        // Initial drawing
        document.addEventListener('DOMContentLoaded', () => {
            drawCaptcha(currentCaptchaCode);
             // Clear CAPTCHA input on page load for security
             captchaInput.value = '';
        });

        // Google Sign-In handler
        function handleCredentialResponse(response) {
           console.log("Encoded JWT ID token: " + response.credential);
           // Send this token to your server for validation
           // Use fetch API to send the token to a PHP script
           fetch('verify_google_login.php', {
               method: 'POST',
               headers: {
                   'Content-Type': 'application/x-www-form-urlencoded' // Or 'application/json' if you prefer
               },
               body: 'credential=' + response.credential
           })
           .then(response => response.json()) // Assuming your PHP returns JSON
           .then(data => {
               if (data.success) {
                   // Redirect on successful login/registration
                   window.location.href = data.redirect || '../beranda/index.php'; // Redirect to provided URL or default
               } else {
                   // Display error message
                   alert('Google login failed: ' + (data.message || 'Unknown error')); // Simple alert for now
                   // Optionally display the error message on the page
                   const errorMessageElement = document.querySelector('.error-message');
                   if (errorMessageElement) {
                       errorMessageElement.innerText = 'Google login failed: ' + (data.message || 'Unknown error');
                       errorMessageElement.style.display = 'block';
                   } else {
                        const newErrorMessageElement = document.createElement('p');
                        newErrorMessageElement.className = 'error-message';
                        newErrorMessageElement.innerText = 'Google login failed: ' + (data.message || 'Unknown error');
                        document.querySelector('.form-container').prepend(newErrorMessageElement);
                   }

               }
           })
           .catch(error => {
               console.error('Error verifying Google token:', error);
               alert('An error occurred during Google login.');
           });
        }
    </script>
</body>
</html>
```

---

**`autentikasi/form-register.php`**

```php
<?php
// autentikasi/form-register.php
require_once '../includes/config.php'; // Include config.php and database functions

// Redirect if already authenticated
if (isAuthenticated()) {
    header('Location: ../beranda/index.php');
    exit;
}
// --- Logika generate CAPTCHA (Server-side) ---
// Generate CAPTCHA new if not set or needed after POST error
if (!isset($_SESSION['captcha_code']) || isset($_SESSION['error_message'])) {
     $_SESSION['captcha_code'] = generateRandomString(6);
}

// --- Proses Form Register ---
$error_message = null;
$success_message = null;

// Retrieve input values from session if redirected back due to error (improves UX)
// Note: Password fields are NOT pre-filled for security
$full_name = $_SESSION['register_full_name'] ?? '';
$username = $_SESSION['register_username'] ?? '';
$age_input = $_SESSION['register_age'] ?? '';
$gender = $_SESSION['register_gender'] ?? '';
$email = $_SESSION['register_email'] ?? '';
$agree = $_SESSION['register_agree'] ?? false;

// Get messages from session and clear them
if (isset($_SESSION['error_message'])) {
    $error_message = $_SESSION['error_message'];
    unset($_SESSION['error_message']);
}
if (isset($_SESSION['success_message'])) {
    $success_message = $_SESSION['success_message'];
    unset($_SESSION['success_message']);
}

// Clear stored inputs from session AFTER retrieving messages
unset($_SESSION['register_full_name']);
unset($_SESSION['register_username']);
unset($_SESSION['register_age']);
unset($_SESSION['register_gender']);
unset($_SESSION['register_email']);
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


    // --- Validasi CAPTCHA (Server-side) ---
    // Check CAPTCHA only if other required fields are present
    if (empty($errors)) {
         if (!isset($_SESSION['captcha_code']) || strtolower($captcha_input_post) !== strtolower($_SESSION['captcha_code'])) {
             $errors[] = 'Invalid CAPTCHA.';
             // CAPTCHA is regenerated at the top of the file if there are errors
         } else {
             // CAPTCHA valid, unset it immediately to prevent reuse
             unset($_SESSION['captcha_code']);
         }
    }


    // If there are validation errors, store them in session and redirect
    if (!empty($errors)) {
        $_SESSION['error_message'] = implode('<br>', $errors);
        header('Location: form-register.php');
        exit;
    }

    // --- If all validations pass, proceed to DB checks and INSERT ---

    // Check if username or email already exists
    try {
        // Use the refactored function from database.php
        if (isUsernameOrEmailExists($username_post, $email_post)) {
            $_SESSION['error_message'] = 'Username or email is already registered.';
             // CAPTCHA is regenerated at the top of the file if there are errors
            header('Location: form-register.php');
            exit;
        }

        // Save new user to database using the createUser function
        // Password hashing is now handled inside createUser
        $inserted = createUser(
            $full_name_post,
            $username_post,
            $email_post,
            $password_post, // Pass raw password, function will hash it
            $age, // Use the validated integer age
            $gender_post
            // profile_image, bio, google_id are null by default or not provided here
        );


        if ($inserted) {
             // Get the ID of the newly created user
            $userId = $pdo->lastInsertId();
            $_SESSION['user_id'] = $userId;

            // Regenerate session ID after successful registration
            session_regenerate_id(true);

            $_SESSION['success_message'] = 'Registration successful! Welcome, ' . htmlspecialchars($username_post) . '!';

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
            $_SESSION['error_message'] = 'Failed to create user account.';
             // CAPTCHA is regenerated at the top of the file if there are errors
            header('Location: form-register.php');
            exit;
        }


    } catch (PDOException $e) {
        error_log("Database error during registration: " . $e->getMessage());
        $_SESSION['error_message'] = 'An internal error occurred during registration. Please try again.';
         // CAPTCHA is regenerated at the top of the file if there are errors
        header('Location: form-register.php');
        exit;
    }
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
    <!-- Google Sign-In -->
    <script src="https://accounts.google.com/gsi/client" async defer></script>
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

         <div class="form-link separator">OR</div>

        <!-- Google Sign-In Button -->
         <div class="google-signin-container">
             <!-- The following div is provided by Google Identity Services -->
             <div id="g_id_onload"
                  data-client_id="<?php echo GOOGLE_CLIENT_ID; ?>"
                  data-callback="handleCredentialResponse"
                  data-auto_prompt="false">
             </div>
             <div class="g_id_signin"
                  data-type="standard"
                  data-size="large"
                  data-theme="filled_blue"
                  data-text="signup_with"
                  data-shape="rectangular"
                  data-logo_alignment="left">
             </div>
         </div>


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
        const captchaCanvas = document.getElementById('captchaCanvas');
        const agreeCheckbox = document.getElementById('agree-checkbox');
        const registerSubmitBtn = document.getElementById('register-submit-btn');


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
            } catch (error) {
                console.error("Error generating CAPTCHA:", error);
                // Optionally display an error message on the page
            }
        }

        // Initial drawing
        document.addEventListener('DOMContentLoaded', () => {
            drawCaptcha(currentCaptchaCode);

             // Set initial state of the Register button based on the agreement checkbox
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
            const captchaInput = document.getElementById('captchaInput').value; // Get current input value

            // Basic password match check
            if (password !== confirmPassword) {
                alert('Password and Confirm Password do not match!');
                return false; // Prevent form submission
            }

             // Check agreement
             if (!agreeCheckbox.checked) {
                 alert('You must agree to the User Agreement.');
                 return false; // Prevent form submission
            }

            // Client-side CAPTCHA check (optional, server-side is mandatory)
            // This provides immediate feedback but is not the primary security check
            // if (captchaInput.toLowerCase() !== currentCaptchaCode.toLowerCase()) {
            //      alert('Invalid CAPTCHA!');
            //      generateCaptcha(); // Regenerate CAPTCHA on client-side failure
            //      return false; // Prevent form submission
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

        // Google Sign-In handler
         function handleCredentialResponse(response) {
            console.log("Encoded JWT ID token: " + response.credential);
            // Send this token to your server for validation
            fetch('verify_google_login.php', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded'
                },
                body: 'credential=' + response.credential
            })
            .then(response => response.json()) // Assuming your PHP returns JSON
            .then(data => {
                if (data.success) {
                    // Redirect on successful login/registration
                    window.location.href = data.redirect || '../beranda/index.php';
                } else {
                    // Display error message
                     alert('Google registration failed: ' + (data.message || 'Unknown error')); // Simple alert for now
                     const errorMessageElement = document.querySelector('.error-message');
                     if (errorMessageElement) {
                         errorMessageElement.innerText = 'Google registration failed: ' + (data.message || 'Unknown error');
                         errorMessageElement.style.display = 'block';
                     } else {
                          const newErrorMessageElement = document.createElement('p');
                          newErrorMessageElement.className = 'error-message';
                          newErrorMessageElement.innerText = 'Google registration failed: ' + (data.message || 'Unknown error');
                          document.querySelector('.form-container').prepend(newErrorMessageElement);
                     }
                }
            })
            .catch(error => {
                console.error('Error verifying Google token:', error);
                alert('An error occurred during Google registration.');
            });
         }
    </script>
</body>
</html>
```

---

**`autentikasi/generate_captcha.php`** (No change needed)

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

**`autentikasi/logout.php`** (No change needed - it was already correct)

```php
<?php
// logout.php
require_once '../includes/config.php'; // Include config.php

// Ensure session is active (should be handled by config.php)
if (session_status() == PHP_SESSION_NONE) {
    session_start();
}

// Unset specific user-related session variables
unset($_SESSION['user_id']);
unset($_SESSION['google_login']); // Unset google login marker

// Optionally clear all session variables (might affect other non-user session data)
// $_SESSION = array();

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

**`autentikasi/style.css`** (Added style for Google Sign-In separator)

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
    display: flex; /* Use flex for internal layout */
    flex-direction: column; /* Stack content */
    gap: 15px; /* Space between elements */
}

/* Class added by JS to show the form */
.form-container.show {
    opacity: 1;
    transform: translateY(0);
}


h2 {
    text-align: center;
    margin-bottom: 5px; /* Adjust margin due to form container gap */
    color: #00e4f9; /* Teal color */
}

/* General input group styling */
.input-group {
    margin-bottom: 0; /* Adjust margin due to form container gap */
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
    margin-top: 0; /* Adjust margin due to form container gap */
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
    margin-top: 0; /* Adjust margin due to form container gap */
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

/* Separator for OR */
.form-link.separator {
    display: block;
    width: 100%;
    text-align: center;
    border-bottom: 1px solid #363636;
    line-height: 0.1em;
    margin: 10px 0 20px; /* Adjust spacing */
    color: #888;
    font-size: 12px;
}
.form-link.separator span {
    background:#2c5364;
    padding:0 10px;
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
    margin-bottom: 0; /* Adjust margin due to form container gap */
    margin-top: 10px; /* Add some space after agreement button */
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
    margin: 10px 0; /* Keep some margin for alerts */
    text-align: center;
    background-color: rgba(255, 107, 107, 0.1);
    padding: 8px;
    border-radius: 4px;
    border: 1px solid rgba(255, 107, 107, 0.4);
}

.success-message {
    color: #6bff6b; /* Greenish */
    font-size: 14px;
    margin: 10px 0; /* Keep some margin for alerts */
    text-align: center;
    background-color: rgba(107, 255, 107, 0.1);
    padding: 8px;
    border-radius: 4px;
    border: 1px solid rgba(107, 255, 107, 0.4);
}

/* Google Sign-In Container */
.google-signin-container {
     display: flex;
     justify-content: center;
     margin-top: 10px;
     margin-bottom: 10px;
}

/* Specific Google Button styles (if needed, though Google GSI handles most) */
/* You might override padding or margin here if the default doesn't fit */


```

---

**`autentikasi/verify_google_login.php`** (New file for server-side Google token verification)

```php
<?php
// autentikasi/verify_google_login.php
require_once '../includes/config.php'; // Include config.php for session, DB, and GOOGLE_CLIENT_ID

// Set response header to JSON
header('Content-Type: application/json');

// Ensure it's a POST request
if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    echo json_encode(['success' => false, 'message' => 'Invalid request method']);
    exit;
}

// Get the ID token from the POST data
$id_token = $_POST['credential'] ?? '';

if (empty($id_token)) {
    echo json_encode(['success' => false, 'message' => 'No Google credential received']);
    exit;
}

// --- Verify the ID token ---
// Use Google's tokeninfo endpoint for simple server-side validation
$url = 'https://oauth2.googleapis.com/tokeninfo?id_token=' . $id_token;
$response = file_get_contents($url);

if ($response === false) {
    error_log("Failed to fetch Google tokeninfo for token: " . substr($id_token, 0, 30) . '...');
    echo json_encode(['success' => false, 'message' => 'Failed to verify Google token with Google.']);
    exit;
}

$token_info = json_decode($response, true);

// Check if token verification was successful and valid for your app
if (!isset($token_info['aud']) || $token_info['aud'] !== GOOGLE_CLIENT_ID) {
     // Check if the audience matches your client ID
     error_log("Google token verification failed: Invalid audience. Received: " . ($token_info['aud'] ?? 'N/A') . ", Expected: " . GOOGLE_CLIENT_ID);
     echo json_encode(['success' => false, 'message' => 'Google token verification failed: Invalid audience.']);
     exit;
}

// Basic checks (issuer, expiry - although tokeninfo does this)
// 'iss' should be accounts.google.com or https://accounts.google.com
if (!isset($token_info['iss']) || !in_array($token_info['iss'], ['accounts.google.com', 'https://accounts.google.com'])) {
     error_log("Google token verification failed: Invalid issuer. Received: " . ($token_info['iss'] ?? 'N/A'));
     echo json_encode(['success' => false, 'message' => 'Google token verification failed: Invalid issuer.']);
     exit;
}

// Check expiry (tokeninfo usually handles this implicitly, but explicit check is safer)
if (!isset($token_info['exp']) || $token_info['exp'] < time()) {
     error_log("Google token verification failed: Token expired for Google ID: " . ($token_info['sub'] ?? 'N/A'));
     echo json_encode(['success' => false, 'message' => 'Google token verification failed: Token expired.']);
     exit;
}


// Extract user information from the verified token
$google_id = $token_info['sub'] ?? null; // 'sub' is the unique Google User ID
$email = $token_info['email'] ?? null;
$full_name = $token_info['name'] ?? null;
$username = $token_info['email'] ?? null; // Use email as a fallback username
$profile_image = $token_info['picture'] ?? null;

if (empty($google_id) || empty($email)) {
     error_log("Google token verification failed: Missing required data (Google ID or email)");
     echo json_encode(['success' => false, 'message' => 'Google token did not contain required user data.']);
     exit;
}


// --- Authenticate or Register the user in your database ---
try {
    // 1. Try to find the user by Google ID
    $user = getUserByGoogleId($google_id);

    if ($user) {
        // User found by Google ID - Log them in
        $_SESSION['user_id'] = $user['user_id'];
        $_SESSION['google_login'] = true; // Optional marker

        // Regenerate session ID
        session_regenerate_id(true);

        // Redirect to intended URL or beranda
        $redirect_url = '../beranda/index.php';
        if (isset($_SESSION['intended_url'])) {
            $redirect_url = $_SESSION['intended_url'];
            unset($_SESSION['intended_url']);
        }

        echo json_encode(['success' => true, 'redirect' => $redirect_url]);
        exit;

    } else {
        // User not found by Google ID - Try to find by email
        $user_by_email = getUserByEmail($email);

        if ($user_by_email) {
            // User found by email - Link Google ID to this existing account
            // This might indicate they previously registered with email/password
            // or logged in with Google before the google_id column was added.
            // If they logged in before the google_id column, updating ensures the link.
            // If they have a local account with the same email, you might want
            // a more complex flow (e.g., prompt them to link accounts, require password),
            // but for simplicity, we'll link it automatically assuming email ownership implies account ownership.

            // Check if the existing account already has a google_id (shouldn't happen if we got here)
            if (!empty($user_by_email['google_id']) && $user_by_email['google_id'] !== $google_id) {
                 // This scenario is complex: two different Google accounts linked to the same email OR an error.
                 // For now, prevent linking if an ID already exists.
                 error_log("Attempted to link Google ID {$google_id} to user {$user_by_email['user_id']} with email {$email}, but user already has Google ID {$user_by_email['google_id']}.");
                 echo json_encode(['success' => false, 'message' => 'An account with this email is already linked to a different Google account.']);
                 exit;
            }

            // Update existing user with Google ID and potentially profile image
            $update_data = ['google_id' => $google_id];
             if (!empty($profile_image) && empty($user_by_email['profile_image'])) {
                 // Only update profile image if the user doesn't have one already
                 $update_data['profile_image'] = $profile_image;
             }
            updateUser($user_by_email['user_id'], $update_data);

            // Log in the existing user
            $_SESSION['user_id'] = $user_by_email['user_id'];
            $_SESSION['google_login'] = true;

            session_regenerate_id(true);

            $redirect_url = '../beranda/index.php';
            if (isset($_SESSION['intended_url'])) {
                $redirect_url = $_SESSION['intended_url'];
                unset($_SESSION['intended_url']);
            }

            echo json_encode(['success' => true, 'redirect' => $redirect_url]);
            exit;

        } else {
            // User not found by Google ID or email - Register a new account
            // Generate a unique username if email isn't suitable or if you need separate usernames
            // A simple approach: use email prefix, add numbers if it conflicts
            $base_username = explode('@', $email)[0];
            $new_username = $base_username;
            $i = 1;
            // Use the isUsernameOrEmailExists function to check username uniqueness
            while (isUsernameOrEmailExists($new_username, '')) { // Check only username part
                 $new_username = $base_username . $i;
                 $i++;
            }

            // Create the new user account with Google info
            $inserted = createUser(
                $full_name ?? $new_username, // Use full_name if available, else generated username
                $new_username,
                $email,
                null, // No local password for Google users
                null, // Age can be added later
                null, // Gender can be added later
                $google_id,
                $profile_image
            );

            if ($inserted) {
                $newUserId = $pdo->lastInsertId();
                $_SESSION['user_id'] = $newUserId;
                $_SESSION['google_login'] = true;

                session_regenerate_id(true);

                 $_SESSION['success_message'] = 'Google registration successful!';

                $redirect_url = '../beranda/index.php';
                if (isset($_SESSION['intended_url'])) {
                    $redirect_url = $_SESSION['intended_url'];
                    unset($_SESSION['intended_url']);
                }

                echo json_encode(['success' => true, 'redirect' => $redirect_url]);
                exit;
            } else {
                error_log("Database error: Failed to create new user for Google ID: {$google_id}");
                echo json_encode(['success' => false, 'message' => 'Failed to create new user account in database.']);
                exit;
            }
        }

    } catch (PDOException $e) {
        error_log("Database error during Google login/registration: " . $e->getMessage());
        echo json_encode(['success' => false, 'message' => 'An internal database error occurred.']);
        exit;
    }

} else {
    // Should not happen with POST check at the beginning
    echo json_encode(['success' => false, 'message' => 'Invalid request.']);
    exit;
}
?>
```

---

**`acc_page/index.php`** (Fixes bio edit, adds basic profile image upload handler link)

```php
<?php
// acc_page/index.php
require_once '../includes/config.php'; // Include config.php
// auth_helper.php is no longer needed, its functions are in config.php or database.php

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

    // Handle username/display name updates (from modal)
    if (isset($_POST['edit_field']) && isset($_POST['edit_value'])) {
        $field = $_POST['edit_field'];
        $value = trim($_POST['edit_value']);

        if (empty($value)) {
            $errors[] = ucwords($field) . ' cannot be empty!'; // Use field name in error
        } else if ($field === 'username') {
            // Check if username already exists (excluding the current user)
            if (isUsernameOrEmailExists($value, '', $userId)) { // Check only username uniqueness
                $errors[] = 'Username already taken!';
            } else {
                $update_data['username'] = $value;
                $has_update = true;
            }
        } else if ($field === 'displayName') {
            $update_data['full_name'] = $value;
            $has_update = true;
        }
        // Add other editable fields here if needed
        // } else if ($field === 'email') { ... email validation and uniqueness check ... }
        // } else if ($field === 'age') { ... validate int ... }
        // } else if ($field === 'gender') { ... validate enum ... }
    }

     // Note: Profile image upload is handled by a separate AJAX call to upload_profile_image.php


    if (empty($errors) && $has_update) {
        // Update the user in the database using the updateUser function
        if (updateUser($userId, $update_data)) {
            $_SESSION['success_message'] = 'Profile updated successfully!';
             // Refresh user data after update
            $user = getAuthenticatedUser(); // Re-fetch user details from DB
             $success = true;
        } else {
            // This might happen if execute fails for some reason (e.g., DB constraint violation not caught by checks)
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
     <link rel="stylesheet" href="../review/styles.css"> <!-- Keep review styles for movie grid -->
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
                        <label for="profile_image_upload" class="edit-icon">
                            <i class="fas fa-camera" title="Change Profile Image"></i>
                        </label>
                        <!-- The actual file input, hidden -->
                        <input type="file" id="profile_image_upload" name="profile_image" accept="image/*" style="display: none;" onchange="uploadProfileImage(this)">
                    </div>
                    <div class="profile-details">
                        <h1>
                            <span id="displayName" onclick="showEditModal('displayName', '<?php echo htmlspecialchars($user['full_name'] ?? ''); ?>')"><?php echo htmlspecialchars($user['full_name'] ?? 'N/A'); ?></span>
                            <i class="fas fa-pen edit-icon"></i>
                        </h1>
                        <p class="username">
                            @<span id="username" onclick="showEditModal('username', '<?php echo htmlspecialchars($user['username']); ?>')"><?php echo htmlspecialchars($user['username']); ?></span>
                            <i class="fas fa-pen edit-icon"></i>
                        </p>
                         <p class="user-meta"><?php echo htmlspecialchars($user['age'] ?? 'N/A'); ?> | <?php echo htmlspecialchars($user['gender'] ?? 'N/A'); ?></p>
                         <!-- Add more fields like email, age, gender here if you implement modals for them -->
                          <!-- <p class="user-meta">Email: <span id="email" onclick="showEditModal('email', '<?php // echo htmlspecialchars($user['email']); ?>')"><?php // echo htmlspecialchars($user['email']); ?></span> <i class="fas fa-pen edit-icon"></i></p> -->
                    </div>
                </div>
                <div class="about-me">
                    <h2>ABOUT ME:</h2>
                    <!-- Bio section - make it editable -->
                    <div class="about-content" id="bio" onclick="enableBioEdit()">
                         <?php
                            $bio_text = $user['bio'] ?? '';
                            // Display placeholder if bio is empty
                            if (empty(trim($bio_text))) {
                                echo 'Click to add bio...';
                            } else {
                                // Use nl2br and htmlspecialchars for displaying bio content
                                echo nl2br(htmlspecialchars($bio_text));
                            }
                         ?>
                    </div>
                </div>
            </div>
            <div class="posts-section">
                <h2>Movies I Uploaded</h2>
                <div class="movies-grid review-grid">
                    <?php if (!empty($uploadedMovies)): ?>
                        <?php foreach ($uploadedMovies as $movie): ?>
                            <div class="movie-card" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'"> <!-- Make the card clickable -->
                                <div class="movie-poster">
                                    <img src="<?php echo htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                    <div class="movie-actions">
                                         <!-- Link to edit page -->
                                         <a href="../manage/edit.php?id=<?php echo $movie['movie_id']; ?>" class="action-btn" title="Edit Movie"><i class="fas fa-edit"></i></a>
                                         <!-- Delete form (should point to manage/indeks.php handler) -->
                                         <form action="../manage/indeks.php" method="POST" onsubmit="return confirm('Are you sure you want to delete &quot;<?php echo htmlspecialchars($movie['title']); ?>&quot;? This cannot be undone.');">
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
        // Function to handle profile image upload
        function uploadProfileImage(input) {
            if (input.files && input.files[0]) {
                const formData = new FormData();
                formData.append('profile_image', input.files[0]);
                // We don't need update_profile=1 here as upload_profile_image.php is a dedicated script

                fetch('upload_profile_image.php', { // Fetch to the separate upload script
                    method: 'POST',
                    body: formData
                })
                .then(response => {
                     if (!response.ok) {
                         // Handle HTTP errors (e.g., 400, 500)
                         return response.text().then(text => { throw new Error('HTTP error! Status: ' + response.status + ' - ' + text); });
                     }
                     return response.json();
                 })
                .then(data => {
                    if (data.success) {
                        // Update image preview src (if it exists)
                        const profileImg = document.querySelector('.profile-image img');
                        if (profileImg) {
                             profileImg.src = data.image_url;
                        }
                        // Show success message (can use the alert div)
                        displayMessage('Profile image updated successfully!', 'success');
                        // Optionally reload the page to ensure sidebar/header updates
                        // window.location.reload(); // Might be jarring, rely on JS update + message
                    } else {
                        // Show error message
                        displayMessage(data.message || 'Failed to update profile image', 'error');
                    }
                })
                .catch(error => {
                    console.error('Error uploading profile image:', error);
                    displayMessage('An error occurred while uploading profile image: ' + error.message, 'error');
                });
            }
        }

         // Helper function to display messages in the alert divs
         function displayMessage(message, type) {
             const alertContainer = document.querySelector('main.main-content');
             // Remove any existing alerts first
             const existingAlerts = alertContainer.querySelectorAll('.alert');
             existingAlerts.forEach(alert => alert.remove());

             const alertDiv = document.createElement('div');
             alertDiv.className = `alert ${type}`;
             alertDiv.innerHTML = message; // Use innerHTML because message might contain <br> from PHP errors

             // Insert before the profile header
             const profileHeader = document.querySelector('.profile-header');
             if (profileHeader) {
                 alertContainer.insertBefore(alertDiv, profileHeader);
             } else {
                 // Fallback if profile header not found
                 alertContainer.prepend(alertDiv);
             }

             // Optional: Automatically hide message after a few seconds
             // setTimeout(() => {
             //     alertDiv.remove();
             // }, 5000); // Hide after 5 seconds
         }


        // JavaScript for bio editing
        function enableBioEdit() {
            const bioElement = document.getElementById('bio');
             // Get current content, handle placeholder
            const currentText = bioElement.textContent.trim();
             const initialBioValue = (currentText === 'Click to add bio...') ? '' : currentText;


            // Check if already in edit mode
            if (bioElement.querySelector('form')) { // Check for the form element instead of textarea
                return;
            }

            const textarea = document.createElement('textarea');
            textarea.className = 'edit-input bio-input';
            textarea.name = 'bio'; // Crucial for POST data
            textarea.value = initialBioValue;
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
            form.style.display = 'flex'; // Use flex to layout textarea and buttons
            form.style.flexDirection = 'column'; // Stack them

            const hiddenUpdateInput = document.createElement('input');
            hiddenUpdateInput.type = 'hidden';
            hiddenUpdateInput.name = 'update_profile'; // Identifier for the POST handler
            hiddenUpdateInput.value = '1';

            // The textarea itself will send the 'bio' data because it has the 'name="bio"' attribute

            // Replace content with form
            bioElement.innerHTML = ''; // Clear previous content
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
                // Restore original content with preserved line breaks and placeholder logic
                bioElement.innerHTML = formatBioForDisplay(initialBioValue);
            };

             // Add keydown listener to textarea for saving on Ctrl+Enter (optional)
            textarea.addEventListener('keydown', function(event) {
                 // Check if Enter key is pressed and Ctrl/Cmd is held
                 if (event.key === 'Enter' && (event.ctrlKey || event.metaKey)) {
                     event.preventDefault(); // Prevent default newline
                     form.submit(); // Submit the form
                 }
            });
        }

         // Helper function for displaying bio (combines nl2br and htmlspecialchars)
         function formatBioForDisplay(str) {
             if (typeof str !== 'string') return str;
             const escapedStr = htmlspecialchars(str);
             // If empty after trim, show placeholder
             if (escapedStr.trim() === '') {
                 return 'Click to add bio...';
             }
             // Replace \r\n, \r, or \n with <br>
             return escapedStr.replace(/(?:\r\n|\r|\n)/g, '<br>');
         }


          // Helper function for HTML escaping (client-side) - already exists, ensuring it's here
         function htmlspecialchars(str) {
             if (typeof str !== 'string') return str;
             // Create a temporary DOM element to leverage browser's escaping
             const div = document.createElement('div');
             div.appendChild(document.createTextNode(str));
             return div.innerHTML;

             // // Or manually, be careful with entity names
             // return str.replace(/&/g, '&amp;')
             //           .replace(/</g, '&lt;')
             //           .replace(/>/g, '&gt;')
             //           .replace(/"/g, '&quot;')
             //           .replace(/'/g, '&#039;');
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

         /* Adjust movie-card cursor */
         .movies-grid.review-grid .movie-card {
             cursor: pointer; /* Indicate clickability */
         }


         /* Styles for the display bio area */
        #bio {
            cursor: pointer;
            transition: background-color 0.3s;
             padding: 10px;
             border-radius: 8px;
             line-height: 1.6;
             color: #ccc;
             white-space: pre-wrap; /* Preserve line breaks */
             min-height: 100px; /* Ensure click area is large enough */
             display: block; /* Make it a block element */
        }

        #bio:hover {
            background-color: #2a2a2a;
        }

        #bio form { /* Style the form wrapper when in edit mode */
            padding: 0;
            margin: 0;
            background: none; /* Remove background from form wrapper */
        }

        /* Style for the bio textarea in edit mode */
        #bio textarea {
             background-color: #363636; /* Darker background for input */
             border: 1px solid #00ffff;
             border-radius: 4px;
             color: #ffffff;
             padding: 8px 12px;
             font-size: inherit;
             width: 100%;
             min-height: 100px;
             resize: vertical;
             outline: none;
             font-family: inherit;
             transition: border-color 0.3s, box-shadow 0.3s;
             line-height: 1.5;
        }
         #bio textarea:focus {
             border-color: #00cccc;
             box-shadow: 0 0 0 2px rgba(0, 255, 255, 0.2);
         }


     </style>
    <!-- Edit Modal -->
    <div id="editModal" class="modal">
        <div class="modal-content">
            <span class="close">&times;</span>
            <h2>Edit <span id="editFieldName"></span></h2>
            <form id="editForm" method="POST" action="index.php">
                <input type="text" id="editInput" name="edit_value" required>
                <input type="hidden" id="editField" name="edit_field">
                <input type="hidden" name="update_profile" value="1">
                <div class="modal-buttons">
                    <button type="button" class="btn-cancel-bio" onclick="closeEditModal()">Cancel</button>
                    <button type="submit" class="btn-save-bio">Save</button>
                </div>
            </form>
        </div>
    </div>

    <script>
        // Modal functions for Username/Display Name
        function showEditModal(field, currentValue) {
            const modal = document.getElementById('editModal');
            const editInput = document.getElementById('editInput');
            const editField = document.getElementById('editField');
            const editFieldName = document.getElementById('editFieldName');

            editInput.value = currentValue;
            editField.value = field;
            editFieldName.textContent = field === 'username' ? 'Username' : 'Display Name';

            modal.style.display = 'block';
            editInput.focus(); // Set focus to the input field
        }

        function closeEditModal() {
            document.getElementById('editModal').style.display = 'none';
             // Clear input and field name when closing
            document.getElementById('editInput').value = '';
            document.getElementById('editField').value = '';
            document.getElementById('editFieldName').textContent = '';
        }

        // Close modal when clicking outside
        window.onclick = function(event) {
            const modal = document.getElementById('editModal');
            if (event.target === modal) {
                closeEditModal();
            }
        }

        // Close modal when clicking X
        document.querySelector('.close').onclick = closeEditModal;

         // Add keydown listener for Escape key to close modal
         document.addEventListener('keydown', function(event) {
             const modal = document.getElementById('editModal');
             if (event.key === 'Escape' && modal && modal.style.display === 'block') {
                 closeEditModal();
             }
         });

         // Add keydown listener for Enter key to submit modal form
         document.getElementById('editInput').addEventListener('keydown', function(event) {
             if (event.key === 'Enter') {
                 event.preventDefault(); // Prevent default behavior (newline)
                 document.getElementById('editForm').submit();
             }
         });


    </script>

    <style>
        /* Modal styles - Keep existing */
        .modal {
            display: none;
            position: fixed;
            z-index: 1000;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0, 0, 0, 0.5);
        }

        .modal-content {
            background-color: #1a1a1a;
            margin: 15% auto;
            padding: 20px;
            border: 1px solid #00ffff;
            border-radius: 5px;
            width: 80%;
            max-width: 500px;
            position: relative;
            color: white;
        }

        .close {
            color: #00ffff;
            float: right;
            font-size: 28px;
            font-weight: bold;
            cursor: pointer;
        }

        .close:hover {
            color: #00cccc;
        }

        #editInput {
            width: 100%;
            padding: 10px;
            margin: 10px 0;
            border: 1px solid #00ffff;
            border-radius: 4px;
            background-color: #2a2a2a;
            color: white;
        }

        .modal-buttons {
            text-align: right;
            margin-top: 15px;
        }

        /* Hover styles for editable fields */
        #displayName, #username {
            cursor: pointer;
            transition: color 0.3s ease;
        }

        #displayName:hover, #username:hover {
            color: #00ffff;
        }

        /* Ensure edit icon visibility consistency */
        .profile-details h1 .edit-icon,
        .profile-details p .edit-icon {
             font-size: 0.8em;
             margin-left: 8px;
             color: #00ffff;
             opacity: 0.8; /* Always slightly visible */
             transition: opacity 0.3s ease;
        }


         #displayName:hover .edit-icon,
         #username:hover .edit-icon {
             opacity: 1; /* Full opacity on hover */
         }
         /* Add similar styles for email/age/gender if added to the details */


    </style>
</body>
</html>
```

---

**`acc_page/styles.css`** (No change needed)

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
    position: relative;
    background-color: #363636;
}

.profile-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    color: transparent;
    font-size: 0;
    transition: filter 0.3s ease;
}

.profile-image:hover img {
    filter: brightness(70%);
}

.profile-image .edit-icon {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: rgba(0, 0, 0, 0.6);
    color: white;
    padding: 8px;
    border-radius: 50%;
    cursor: pointer;
    opacity: 0;
    transition: opacity 0.3s ease;
}

.profile-image:hover .edit-icon {
    opacity: 1;
}

.profile-image .edit-icon i {
    font-size: 1.2em;
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

**`acc_page/upload_profile_image.php`** (No change needed - it was already correct for AJAX upload)

```php
<?php
// acc_page/upload_profile_image.php
require_once '../includes/config.php';

// Redirect if not authenticated
redirectIfNotAuthenticated();

// Get authenticated user ID
$userId = $_SESSION['user_id'];

// Set response header to JSON
header('Content-Type: application/json');

// Check if file was uploaded
if (!isset($_FILES['profile_image'])) {
    echo json_encode(['success' => false, 'message' => 'No file uploaded']);
    exit;
}

// Define allowed file types and max file size
$allowed_types = ['image/jpeg', 'image/png', 'image/gif', 'image/webp']; // Added webp
$max_size = 5 * 1024 * 1024; // 5MB

$file = $_FILES['profile_image'];

// Validate file type
if (!in_array($file['type'], $allowed_types)) {
    echo json_encode(['success' => false, 'message' => 'Invalid file type. Only JPG, PNG, GIF, WEBP are allowed.']);
    exit;
}

// Validate file size
if ($file['size'] > $max_size) {
    echo json_encode(['success' => false, 'message' => 'File too large. Maximum size is 5MB.']);
    exit;
}

// Create upload directory if it doesn't exist
$upload_dir = __DIR__ . '/../uploads/profile_images/';
if (!file_exists($upload_dir)) {
    mkdir($upload_dir, 0777, true); // Use 0777 temporarily for debugging permissions, consider 0755 or 0775 in production
}

// Generate unique filename
$extension = pathinfo($file['name'], PATHINFO_EXTENSION);
// Use a combination of user ID and unique ID for filename clarity
$filename = 'profile_' . $userId . '_' . uniqid() . '.' . $extension;
$filepath = $upload_dir . $filename;

// Move uploaded file
if (move_uploaded_file($file['tmp_name'], $filepath)) {
    // Update user profile in database
    $web_path = '../uploads/profile_images/' . $filename;
    try {
        // Fetch old profile image path to delete the old file
        $old_user = getUserById($userId); // Use getUserById from database.php
        $old_profile_image = $old_user['profile_image'] ?? null;

        if (updateUser($userId, ['profile_image' => $web_path])) {
             // Delete old file if it exists and is not the default/placeholder
            if (!empty($old_profile_image) && strpos($old_profile_image, 'ui-avatars.com') === false) {
                $old_file_path = __DIR__ . '/../' . $old_profile_image; // Construct absolute path
                if (file_exists($old_file_path)) {
                    @unlink($old_file_path); // Use @ to suppress errors if file is not found/deletable
                }
            }

            echo json_encode([
                'success' => true,
                'image_url' => htmlspecialchars($web_path), // Return HTML escaped URL
                'message' => 'Profile image updated successfully.'
            ]);
        } else {
            // If DB update fails, clean up the uploaded file
            @unlink($filepath);
            echo json_encode(['success' => false, 'message' => 'Failed to update database record for profile image.']);
        }
    } catch (PDOException $e) {
        // If a DB error occurs, clean up the uploaded file
        @unlink($filepath);
        error_log("Database error during profile image update: " . $e->getMessage());
        echo json_encode(['success' => false, 'message' => 'Database error occurred while updating profile image.']);
    }
} else {
    // Handle specific upload errors
    $uploadError = 'Unknown upload error.';
    switch ($_FILES['profile_image']['error']) {
        case UPLOAD_ERR_INI_SIZE:
        case UPLOAD_ERR_FORM_SIZE:
            $uploadError = 'File exceeds maximum allowed size.';
            break;
        case UPLOAD_ERR_PARTIAL:
            $uploadError = 'File was only partially uploaded.';
            break;
        case UPLOAD_ERR_NO_FILE:
            $uploadError = 'No file was uploaded.';
            break;
        case UPLOAD_ERR_NO_TMP_DIR:
            $uploadError = 'Missing a temporary folder.';
            break;
        case UPLOAD_ERR_CANT_WRITE:
            $uploadError = 'Failed to write file to disk.';
            break;
        case UPLOAD_ERR_EXTENSION:
            $uploadError = 'A PHP extension stopped the file upload.';
            break;
    }
    error_log("Profile image move upload failed: " . $uploadError . " - " . $file['name']);
    echo json_encode(['success' => false, 'message' => 'Failed to move uploaded file: ' . $uploadError]);
}
?>
```

---

**`beranda/index.php`** (No change needed)

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
                              <?php if(isAuthenticated()): // Use isAuthenticated ?>
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
         <?php if (isAuthenticated()): // Use isAuthenticated ?>
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

**`beranda/styles.css`** (No change needed)

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
    background-color: #363636;
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

**`favorite/index.php`** (No change needed)

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
                let searchEmptyState = document.querySelector('.search-empty-state'); // Re-select in case it was added/removed

                if (visibleCardCount === 0 && searchTerm !== '') {
                    if (!searchEmptyState) {
                        // Hide the initial empty state if it exists and we are searching
                        if(initialEmptyState) initialEmptyState.style.display = 'none';

                        searchEmptyState = document.createElement('div'); // Create if not exists
                        searchEmptyState.className = 'empty-state search-empty-state full-width';
                        searchEmptyState.innerHTML = `
                            <i class="fas fa-search"></i>
                            <p>No favorites found matching "${htmlspecialchars(searchTerm)}"</p>
                            <p class="subtitle">Try a different search term</p>
                        `;
                        reviewGrid.appendChild(searchEmptyState);
                    } else {
                         // Update text if search empty state already exists
                         searchEmptyState.querySelector('p:first-of-type').innerText = `No favorites found matching "${htmlspecialchars(searchTerm)}"`;
                         searchEmptyState.style.display = 'flex'; // Ensure it's displayed
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
              // Create a temporary DOM element to leverage browser's escaping
             const div = document.createElement('div');
             div.appendChild(document.createTextNode(str));
             return div.innerHTML;
         }
    </script>
</body>
</html>
```

---

**`favorite/styles.css`** (No change needed)

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

**`manage/indeks.php`** (Added delete handling logic)

```php
<?php
// manage/indeks.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated
redirectIfNotAuthenticated();

// Get authenticated user ID
$userId = $_SESSION['user_id'];

// Handle delete movie action
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['delete_movie_id'])) {
    $movieIdToDelete = filter_var($_POST['delete_movie_id'], FILTER_VALIDATE_INT);

    if ($movieIdToDelete) {
        try {
            // Fetch movie details BEFORE deleting to get file paths
            $movieToDelete = getMovieById($movieIdToDelete);

            // Ensure the movie exists and is uploaded by the current user
            if ($movieToDelete && $movieToDelete['uploaded_by'] == $userId) {

                $pdo->beginTransaction();

                // IMPORTANT: Due to ON DELETE CASCADE in the schema, deleting the movie
                // from the 'movies' table will automatically delete associated rows
                // in 'movie_genres', 'reviews', and 'favorites'.

                // Prepare and execute the delete statement for the movie table
                $stmt = $pdo->prepare("DELETE FROM movies WHERE movie_id = ?");
                $stmt->execute([$movieIdToDelete]);

                $pdo->commit();

                // Delete associated files (poster, trailer) from the server AFTER successful DB deletion
                if (!empty($movieToDelete['poster_image'])) {
                     $poster_path = UPLOAD_DIR_POSTERS . $movieToDelete['poster_image'];
                     if (file_exists($poster_path)) {
                         @unlink($poster_path); // Use @ to suppress errors
                     }
                }
                if (!empty($movieToDelete['trailer_file'])) {
                    $trailer_path = UPLOAD_DIR_TRAILERS . $movieToDelete['trailer_file'];
                    if (file_exists($trailer_path)) {
                         @unlink($trailer_path); // Use @ to suppress errors
                    }
                }


                $_SESSION['success_message'] = 'Movie deleted successfully!';
            } else {
                 // Movie not found OR not uploaded by the current user
                $_SESSION['error_message'] = 'Movie not found or you do not have permission to delete it.';
            }

        } catch (PDOException $e) {
            if ($pdo->inTransaction()) {
                $pdo->rollBack();
            }
            error_log("Database error during movie deletion (ID: {$movieIdToDelete}, User: {$userId}): " . $e->getMessage());
            $_SESSION['error_message'] = 'An internal error occurred while deleting the movie.';
        }
    } else {
        $_SESSION['error_message'] = 'Invalid movie ID.';
    }

    header('Location: indeks.php'); // Redirect to prevent form resubmission
    exit;
}


// Fetch movies uploaded by the current user (after potential deletion)
$movies = getUserUploadedMovies($userId);

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
                            <div class="movie-poster" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'"> <!-- Make poster clickable -->
                                <img src="<?php echo htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $movie['poster_image'] ?? '../gambar/placeholder.jpg'); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                <div class="movie-actions">
                                     <!-- Edit Button -->
                                     <a href="edit.php?id=<?php echo $movie['movie_id']; ?>" class="action-btn" title="Edit Movie">
                                         <i class="fas fa-pen"></i>
                                     </a>
                                     <!-- Delete Button -->
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
                let searchEmptyState = document.querySelector('.search-empty-state'); // Re-select

                if (visibleCardCount === 0 && searchTerm !== '') {
                    if (!searchEmptyState) {
                         // Hide the initial empty state if it exists and we are searching
                        if(initialEmptyState) initialEmptyState.style.display = 'none';

                        searchEmptyState = document.createElement('div'); // Create if not exists
                        searchEmptyState.className = 'empty-state search-empty-state full-width';
                        searchEmptyState.innerHTML = `
                            <i class="fas fa-search"></i>
                            <p>No uploaded movies found matching "${htmlspecialchars(searchTerm)}"</p>
                            <p class="subtitle">Try a different search term</p>
                        `;
                        moviesGrid.appendChild(searchEmptyState);
                    } else {
                         // Update text if search empty state already exists
                         searchEmptyState.querySelector('p:first-of-type').innerText = `No uploaded movies found matching "${htmlspecialchars(searchTerm)}"`;
                         searchEmptyState.style.display = 'flex'; // Ensure it's displayed
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
             // Create a temporary DOM element to leverage browser's escaping
             const div = document.createElement('div');
             div.appendChild(document.createTextNode(str));
             return div.innerHTML;
         }
    </script>
</body>
</html>
```

---

**`manage/styles.css`** (No change needed)

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

**`manage/edit.php`** (New file for editing movies)

```php
<?php
// manage/edit.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated
redirectIfNotAuthenticated();

// Get authenticated user ID
$userId = $_SESSION['user_id'];

// Get movie ID from URL
$movieId = filter_var($_GET['id'] ?? null, FILTER_VALIDATE_INT);

// Handle missing or invalid movie ID
if (!$movieId) {
    $_SESSION['error_message'] = 'Invalid movie ID provided for editing.';
    header('Location: indeks.php');
    exit;
}

// Fetch movie details
$movie = getMovieById($movieId);

// Handle movie not found or not uploaded by the current user
if (!$movie || $movie['uploaded_by'] != $userId) {
    $_SESSION['error_message'] = 'Movie not found or you do not have permission to edit it.';
    header('Location: indeks.php');
    exit;
}

// Initialize variables with existing movie data for form pre-filling
$title = $movie['title'];
$summary = $movie['summary'];
$release_date = $movie['release_date'];
$duration_hours = $movie['duration_hours'];
$duration_minutes = $movie['duration_minutes'];
$age_rating = $movie['age_rating'];
$genres = $movie['genres_array']; // Get genres as an array
$trailer_url = $movie['trailer_url'];
$trailer_file = $movie['trailer_file']; // Existing trailer file name
$poster_image = $movie['poster_image']; // Existing poster image name


$error_message = null;
$success_message = null;

// Handle form submission (PUT/POST method emulation if needed, but POST is standard for forms)
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // 1. Collect and Sanitize Input
    // Use $_POST data for updates, fallback to existing movie data if field not submitted
    $title_post = trim($_POST['movie-title'] ?? $title);
    $summary_post = trim($_POST['movie-summary'] ?? $summary);
    $release_date_post = $_POST['release-date'] ?? $release_date;
    // Use filter_var but allow empty string if field wasn't in POST (though form should send it)
    $duration_hours_post = filter_var($_POST['duration-hours'] ?? $duration_hours, FILTER_VALIDATE_INT);
    $duration_minutes_post = filter_var($_POST['duration-minutes'] ?? $duration_minutes, FILTER_VALIDATE_INT);
    $age_rating_post = $_POST['age-rating'] ?? $age_rating;
    $genres_post = $_POST['genre'] ?? []; // Genres are selected checkboxes
    $trailer_url_post = trim($_POST['trailer-link'] ?? $trailer_url);

    $errors = [];
    $update_data = []; // Data array to pass to updateUser function

    // 2. Validate and Prepare Update Data

    // Basic fields
    if (empty($title_post)) $errors[] = 'Movie Title is required.';
    else if ($title_post !== $title) $update_data['title'] = $title_post;

    if (empty($release_date_post)) $errors[] = 'Release Date is required.';
    else if ($release_date_post !== $release_date) $update_data['release_date'] = $release_date_post;

    // Duration validation
    if ($duration_hours_post === false || $duration_hours_post === '' || $duration_hours_post < 0) $errors[] = 'Valid Duration (Hours) is required.';
    else if ($duration_hours_post !== $duration_hours) $update_data['duration_hours'] = $duration_hours_post;

    if ($duration_minutes_post === false || $duration_minutes_post === '' || $duration_minutes_post < 0 || $duration_minutes_post > 59) $errors[] = 'Valid Duration (Minutes) is required (0-59).';
    else if ($duration_minutes_post !== $duration_minutes) $update_data['duration_minutes'] = $duration_minutes_post;

    if (empty($age_rating_post)) $errors[] = 'Age Rating is required.';
    else if ($age_rating_post !== $age_rating) $update_data['age_rating'] = $age_rating_post;


    // Summary (can be empty, but trim)
    if ($summary_post !== $summary) $update_data['summary'] = $summary_post;


    // Genres (handled separately for the movie_genres table)
     $allowed_genres = ['action', 'adventure', 'comedy', 'drama', 'horror', 'supernatural', 'animation', 'sci-fi'];
     foreach ($genres_post as $genre) {
         if (!in_array($genre, $allowed_genres)) {
             $errors[] = 'Invalid genre selected.';
             $genres_post = $genres; // Revert genres to original on error to prevent saving bad data
             break; // Stop checking genres
         }
     }
     // Assume genre update will be handled separately later

    // Trailer Handling
    $new_trailer_file_path = $trailer_file; // Default to existing file
    $new_trailer_url = $trailer_url_post; // Default to submitted URL

    // Check if a new trailer file was uploaded
    if (isset($_FILES['trailer-file']) && $_FILES['trailer-file']['error'] === UPLOAD_ERR_OK) {
         $trailerFile = $_FILES['trailer-file'];
         $allowedTypes = ['video/mp4', 'video/webm', 'video/ogg', 'video/quicktime'];
         $maxFileSize = 50 * 1024 * 1024; // 50MB

         if (!in_array($trailerFile['type'], $allowedTypes)) {
             $errors[] = 'Invalid new trailer file type. Only MP4, WebM, Ogg, MOV are allowed.';
         }
         if ($trailerFile['size'] > $maxFileSize) {
             $errors[] = 'New trailer file is too large. Maximum size is 50MB.';
         }

         // If no file errors yet, process the upload
         if (empty($errors)) {
             $fileExtension = pathinfo($trailerFile['name'], PATHINFO_EXTENSION);
             $newFileName = uniqid('trailer_', true) . '.' . $fileExtension;
             $destination = UPLOAD_DIR_TRAILERS . $newFileName;

             if (move_uploaded_file($trailerFile['tmp_name'], $destination)) {
                 $new_trailer_file_path = $newFileName; // Store new filename
                 $new_trailer_url = null; // Prioritize file, clear URL
             } else {
                 $errors[] = 'Failed to upload new trailer file.';
             }
         }
    } else if (isset($_FILES['trailer-file']) && $_FILES['trailer-file']['error'] !== UPLOAD_ERR_NO_FILE) {
        // Handle specific upload errors for new trailer file
        $errors[] = 'New trailer file upload error: ' . $_FILES['trailer-file']['error'];
    }

     // Check if at least one trailer source is provided (URL or existing/new file)
     if (empty($new_trailer_url) && empty($new_trailer_file_path)) {
         $errors[] = 'Either a Trailer URL or a Trailer File is required.';
     } else {
          // Update trailer fields if they changed
          if ($new_trailer_url !== $trailer_url) $update_data['trailer_url'] = $new_trailer_url;
          if ($new_trailer_file_path !== $trailer_file) $update_data['trailer_file'] = $new_trailer_file_path;
     }


    // Poster Handling
    $new_poster_image_path = $poster_image; // Default to existing poster

    // Check if a new poster image was uploaded
    if (isset($_FILES['movie-poster']) && $_FILES['movie-poster']['error'] === UPLOAD_ERR_OK) {
         $posterFile = $_FILES['movie-poster'];
         $allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
         $maxFileSize = 5 * 1024 * 1024; // 5MB

         if (!in_array($posterFile['type'], $allowedTypes)) {
             $errors[] = 'Invalid new poster file type. Only JPG, PNG, GIF, WEBP are allowed.';
         }
         if ($posterFile['size'] > $maxFileSize) {
             $errors[] = 'New poster file is too large. Maximum size is 5MB.';
         }

         // If no file errors yet, process the upload
         if (empty($errors)) {
             $fileExtension = pathinfo($posterFile['name'], PATHINFO_EXTENSION);
             $newFileName = uniqid('poster_', true) . '.' . $fileExtension;
             $destination = UPLOAD_DIR_POSTERS . $newFileName;

             if (move_uploaded_file($posterFile['tmp_name'], $destination)) {
                 $new_poster_image_path = $newFileName; // Store new filename
             } else {
                 $errors[] = 'Failed to upload new poster file.';
             }
         }
    } else if (isset($_FILES['movie-poster']) && $_FILES['movie-poster']['error'] !== UPLOAD_ERR_NO_FILE) {
        // Handle specific upload errors for new poster file
         $errors[] = 'New poster file upload error: ' . $_FILES['movie-poster']['error'];
    }

     // Update poster field if it changed
     if ($new_poster_image_path !== $poster_image) {
         $update_data['poster_image'] = $new_poster_image_path;
     }


    // 3. If no errors, update database
    if (empty($errors)) {
        try {
            $pdo->beginTransaction();

            $db_update_success = true;

            // Update movie details in the movies table
            if (!empty($update_data)) { // Only call update if there's data to update
                 if (!updateMovie($movieId, $update_data)) {
                     $db_update_success = false;
                 }
            }

            // Update genres - Clear existing and add new ones
            // Only update if the submitted genres are different from the original genres
            // Sorting is needed for comparison as order doesn't matter
            $current_genres_sorted = $genres;
            $submitted_genres_sorted = $genres_post;
            sort($current_genres_sorted);
            sort($submitted_genres_sorted);

            if ($current_genres_sorted !== $submitted_genres_sorted) {
                 // Remove old genres
                 if (!removeMovieGenres($movieId)) {
                     $db_update_success = false;
                 } else {
                     // Add new genres
                     foreach ($genres_post as $genre) {
                         if (!addMovieGenre($movieId, $genre)) {
                             $db_update_success = false; // Record specific genre error if needed, or just fail the whole update
                             break; // Stop adding genres if one fails
                         }
                     }
                 }
            }


            if ($db_update_success) {
                $pdo->commit();

                // Clean up old files AFTER successful DB update
                // Delete old poster if a new one was uploaded
                if ($new_poster_image_path !== $poster_image && !empty($poster_image)) {
                    $old_poster_path = UPLOAD_DIR_POSTERS . $poster_image;
                    if (file_exists($old_poster_path)) {
                        @unlink($old_poster_path);
                    }
                }
                 // Delete old trailer file if a new one was uploaded
                 if ($new_trailer_file_path !== $trailer_file && !empty($trailer_file)) {
                     $old_trailer_path = UPLOAD_DIR_TRAILERS . $trailer_file;
                     if (file_exists($old_trailer_path)) {
                         @unlink($old_trailer_path);
                     }
                 }


                $_SESSION['success_message'] = 'Movie updated successfully!';
                header('Location: indeks.php'); // Redirect to manage page
                exit;

            } else {
                 // DB update failed (either main movie data or genres)
                $pdo->rollBack();
                $errors[] = 'Failed to save changes to the database.'; // Generic error, specific errors were added earlier

                // Clean up newly uploaded files if DB transaction failed
                if ($new_poster_image_path !== $poster_image && file_exists(UPLOAD_DIR_POSTERS . $new_poster_image_path)) {
                     @unlink(UPLOAD_DIR_POSTERS . $new_poster_image_path);
                }
                 if ($new_trailer_file_path !== $trailer_file && file_exists(UPLOAD_DIR_TRAILERS . $new_trailer_file_path)) {
                    @unlink(UPLOAD_DIR_TRAILERS . $new_trailer_file_path);
                }
            }


        } catch (PDOException $e) {
            if ($pdo->inTransaction()) {
                $pdo->rollBack();
            }
            error_log("Database error during movie update (ID: {$movieId}, User: {$userId}): " . $e->getMessage());
            $errors[] = 'An internal error occurred while saving the movie.';

            // Clean up newly uploaded files on exception
            if ($new_poster_image_path !== $poster_image && file_exists(UPLOAD_DIR_POSTERS . $new_poster_image_path)) {
                 @unlink(UPLOAD_DIR_POSTERS . $new_poster_image_path);
            }
             if ($new_trailer_file_path !== $trailer_file && file_exists(UPLOAD_DIR_TRAILERS . $new_trailer_file_path)) {
                @unlink(UPLOAD_DIR_TRAILERS . $new_trailer_file_path);
            }
        }
    }

    // If there were any errors, set the error message
    if (!empty($errors)) {
        $_SESSION['error_message'] = implode('<br>', $errors);
        // Keep existing movie data in variables so form is pre-filled with POST data first, then original movie data
        // No redirect needed as we're already on the edit page
        // Re-fetch movie to get potentially mixed data if some updates succeeded or if $_POST wasn't fully set
        // This is tricky - simpler to just rely on $errors and keep POST data for display
         $title = $title_post; // Update variables to show POST data
         $summary = $summary_post;
         $release_date = $release_date_post;
         $duration_hours = $duration_hours_post;
         $duration_minutes = $duration_minutes_post;
         $age_rating = $age_rating_post;
         $genres = $genres_post;
         $trailer_url = $new_trailer_url;
         $trailer_file = $new_trailer_file_path;
         $poster_image = $new_poster_image_path; // This should be the new name if upload succeeded but DB failed
                                               // But it's better to just show the OLD image on DB failure
         $poster_image = $movie['poster_image']; // Revert poster preview to original on error
         $trailer_file = $movie['trailer_file']; // Revert trailer file preview to original on error
         $trailer_url = $trailer_url_post; // Keep the submitted URL on error

    }
}

// Re-fetch movie data if not a POST request or if POST failed and we didn't explicitly update vars
if ($_SERVER['REQUEST_METHOD'] !== 'POST' || !empty($errors)) {
     // If coming to the page initially OR if POST failed, load/reload from DB
     $movie = getMovieById($movieId); // Re-fetch clean data or updated data
     if (!$movie || $movie['uploaded_by'] != $userId) {
         $_SESSION['error_message'] = 'Movie data could not be reloaded.';
         header('Location: indeks.php');
         exit;
     }
     // Re-assign variables from (potentially updated) $movie data
     $title = $movie['title'];
     $summary = $movie['summary'];
     $release_date = $movie['release_date'];
     $duration_hours = $movie['duration_hours'];
     $duration_minutes = $movie['duration_minutes'];
     $age_rating = $movie['age_rating'];
     $genres = $movie['genres_array'];
     $trailer_url = $movie['trailer_url'];
     $trailer_file = $movie['trailer_file'];
     $poster_image = $movie['poster_image'];
}


// Get messages from session for display
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
    <title>Edit Movie: <?php echo htmlspecialchars($title); ?> - RatingTales</title>
    <link rel="stylesheet" href="styles.css">
    <link rel="stylesheet" href="upload.css"> <!-- Use upload styles for form layout -->
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
                <h1>Edit Movie: <?php echo htmlspecialchars($title); ?></h1>

                 <?php if ($success_message): ?>
                    <div class="alert success"><?php echo htmlspecialchars($success_message); ?></div>
                <?php endif; ?>
                <?php if ($error_message): ?>
                    <div class="alert error"><?php echo $error_message; ?></div>
                <?php endif; ?>

                <form class="upload-form" action="edit.php?id=<?php echo $movieId; ?>" method="post" enctype="multipart/form-data">
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
                                        // Check if the current movie's genres array contains this option
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
                                 <p class="trailer-note" style="margin-top: 10px;">Only one trailer source (URL or File) is needed. Uploading a new file or entering a URL will replace the existing one.</p>
                                <?php if (!empty($trailer_url) || !empty($trailer_file)): ?>
                                    <p class="trailer-note">Current Trailer:
                                         <?php if (!empty($trailer_url)): ?>
                                            <a href="<?php echo htmlspecialchars($trailer_url); ?>" target="_blank">Link</a>
                                         <?php elseif (!empty($trailer_file)): ?>
                                            File: <?php echo htmlspecialchars($trailer_file); ?>
                                         <?php endif; ?>
                                    </p>
                                <?php endif; ?>
                            </div>
                        </div>

                        <div class="form-side">
                            <div class="poster-upload">
                                <label for="movie-poster">Movie Poster</label>
                                <div class="upload-area" id="upload-area">
                                    <i class="fas fa-image"></i>
                                    <p>Click or drag image here</p>
                                     <input type="file" id="movie-poster" name="movie-poster" accept="image/*"> <!-- Not required on edit unless replacing -->
                                    <!-- Display existing poster if available -->
                                    <img id="poster-preview" src="<?php echo !empty($poster_image) ? htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $poster_image) : '#'; ?>" alt="Poster Preview" style="<?php echo !empty($poster_image) ? 'display: block; object-fit: cover;' : 'display: none;'; ?>">
                                </div>
                                <p class="trailer-note" style="margin-top: 5px;">(Recommended: Aspect Ratio 2:3, Max 5MB)</p>
                                <?php if (!empty($poster_image)): ?>
                                    <p class="trailer-note">Current Poster: <?php echo htmlspecialchars($poster_image); ?></p>
                                <?php endif; ?>
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
                        <button type="submit" class="submit-btn">Save Changes</button>
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

        // Initial display based on existing poster
        const hasExistingPoster = posterPreview.src && posterPreview.src !== window.location.href + '#'; // Check if src is set and not empty hash
        if (hasExistingPoster) {
             uploadAreaIcon.style.display = 'none'; // Hide icon
             uploadAreaText.style.display = 'none'; // Hide text
             posterPreview.style.objectFit = 'cover'; // Use cover for existing
        } else {
             posterPreview.style.display = 'none'; // Ensure hidden if no poster
             uploadAreaIcon.style.display = ''; // Show icon
             uploadAreaText.style.display = ''; // Show text
        }


        posterInput.addEventListener('change', function() {
            const file = this.files[0];
            if (file) {
                const reader = new FileReader();
                reader.onload = function(e) {
                    posterPreview.src = e.target.result;
                    posterPreview.style.display = 'block';
                    uploadAreaIcon.style.display = 'none'; // Hide icon
                    uploadAreaText.style.display = 'none'; // Hide text
                     posterPreview.style.objectFit = 'contain'; // Use contain for new upload preview
                }
                reader.readAsDataURL(file); // Read the file as a data URL
            } else {
                // If file input is cleared (not typical for file inputs but good fallback)
                // Revert to existing poster if any, or show empty state
                 if (hasExistingPoster) {
                      // Restore existing poster preview state
                      posterPreview.src = "<?php echo !empty($poster_image) ? htmlspecialchars(WEB_UPLOAD_DIR_POSTERS . $poster_image) : '#'; ?>";
                      posterPreview.style.display = 'block';
                      uploadAreaIcon.style.display = 'none';
                      uploadAreaText.style.display = 'none';
                      posterPreview.style.objectFit = 'cover';
                 } else {
                      // Show empty state
                      posterPreview.src = '#';
                      posterPreview.style.display = 'none';
                      uploadAreaIcon.style.display = '';
                      uploadAreaText.style.display = '';
                 }
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
             // Disable file input only if the URL input has a non-empty value after trimming
             trailerFileInput.disabled = this.value.trim() !== '';
             if (trailerFileInput.disabled) {
                 trailerFileInput.value = ''; // Clear file input if URL is entered
             }
         });

         trailerFileInput.addEventListener('change', function() {
             // Disable URL input only if a file has been selected
             trailerLinkInput.disabled = this.files.length > 0;
             if (trailerLinkInput.disabled) {
                 trailerLinkInput.value = ''; // Clear URL input if file is selected
             }
         });

         // Initial check on page load (important for pre-filled values)
         document.addEventListener('DOMContentLoaded', () => {
             // Disable the counterpart input if either URL or File is already set
             if (trailerLinkInput.value.trim() !== '') {
                 trailerFileInput.disabled = true;
             } else if (trailerFileInput.files.length > 0 || "<?php echo !empty($trailer_file); ?>" === '1') {
                 // Check if a file is currently selected OR if a trailer_file exists from PHP
                 trailerLinkInput.disabled = true;
             }
              // Note: trailerFileInput.files.length > 0 check here might not work if file input was cleared client-side but not submitted
              // Relying on PHP value is more reliable for initial state from existing data.
              if ("<?php echo !empty($trailer_file); ?>" === '1') {
                   trailerLinkInput.disabled = true;
              } else if (trailerLinkInput.value.trim() !== '') {
                   trailerFileInput.disabled = true;
              }

         });

         // Helper function for HTML escaping (client-side) - good practice if displaying user input again
         function htmlspecialchars(str) {
             if (typeof str !== 'string') return str;
              // Create a temporary DOM element to leverage browser's escaping
             const div = document.createElement('div');
             div.appendChild(document.createTextNode(str));
             return div.innerHTML;
         }
    </script>
</body>
</html>
```

---

**`review/index.php`** (No change needed)

```php
<?php
// review/index.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated (optional, but good practice for review section)
// redirectIfNotAuthenticated(); // Removed redirectIfNotAuthenticated to allow viewing without login

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
                <?php if (isAuthenticated()): ?>
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favourites</span></a></li>
                 <?php endif; ?>
                <li class="active"><a href="#"><i class="fas fa-star"></i> <span>Review</span></a></li>
                <?php if (isAuthenticated()): ?>
                <li><a href="../manage/indeks.php"><i class="fas fa-film"></i> <span>Manage</span></a></li>
                <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li>
                 <?php endif; ?>
            </ul>
            <div class="bottom-links">
                <ul>
                     <?php if (isAuthenticated()): ?>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li>
                    <?php else: ?>
                    <li><a href="../autentikasi/form-login.php"><i class="fas fa-sign-in-alt"></i> <span>Login</span></a></li>
                    <?php endif; ?>
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
                let searchEmptyState = document.querySelector('.search-empty-state'); // Re-select

                if (visibleCardCount === 0 && searchTerm !== '') {
                    if (!searchEmptyState) {
                         // Hide the initial empty state if it exists and we are searching
                        if(initialEmptyState) initialEmptyState.style.display = 'none';

                        searchEmptyState = document.createElement('div'); // Create if not exists
                        searchEmptyState.className = 'empty-state search-empty-state full-width';
                        emptyState.innerHTML = `
                            <i class="fas fa-search"></i>
                            <p>No movies found matching "${htmlspecialchars(searchTerm)}"</p>
                            <p class="subtitle">Try a different search term</p>
                        `;
                        reviewGrid.appendChild(searchEmptyState);
                    } else {
                         // Update text if search empty state already exists
                         searchEmptyState.querySelector('p:first-of-type').innerText = `No movies found matching "${htmlspecialchars(searchTerm)}"`;
                         searchEmptyState.style.display = 'flex'; // Ensure it's displayed
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
              // Create a temporary DOM element to leverage browser's escaping
             const div = document.createElement('div');
             div.appendChild(document.createTextNode(str));
             return div.innerHTML;
         }
    </script>
</body>
</html>
```

---

**`review/movie-details.php`** (Fixes trailer playback, loads user's existing review)

```php
<?php
// review/movie-details.php
require_once '../includes/config.php'; // Include config.php

// Redirect if not authenticated (optional, but good practice for review page actions)
// redirectIfNotAuthenticated(); // Removed redirect for public viewing, actions will check login state

// Get authenticated user details (if logged in)
$userId = null;
$user = null;
if (isAuthenticated()) {
    $userId = $_SESSION['user_id'];
    $user = getAuthenticatedUser(); // Fetch user details for comments (username, profile_image)
}


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

// Fetch the current user's review for this movie (if logged in)
$userReview = null;
if ($userId) {
    $userReview = getUserReviewForMovie($movieId, $userId);
}


// Check if the movie is favorited by the current user (if logged in)
$isFavorited = $userId ? isMovieFavorited($movieId, $userId) : false;

// Handle comment and rating submission
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['submit_review'])) {
    // Check if logged in to submit review
    if (!$userId) {
         $_SESSION['error_message'] = 'You must be logged in to leave a review.';
         // Store intended URL before redirecting to login
         $_SESSION['intended_url'] = $_SERVER['REQUEST_URI'];
         header('Location: ../autentikasi/form-login.php');
         exit;
    }

    $commentText = trim($_POST['comment'] ?? '');
    $submittedRating = filter_var($_POST['rating'] ?? null, FILTER_VALIDATE_FLOAT);

     // Basic validation for rating
     if ($submittedRating === false || $submittedRating < 0.5 || $submittedRating > 5) {
          $_SESSION['error_message'] = 'Please provide a valid rating (0.5 to 5).';
     } else {
          // Allow empty comment with rating, but trim it
          if (createReview($movieId, $userId, $submittedRating, $commentText)) {
              $_SESSION['success_message'] = 'Your review has been submitted!';
              // Update the $userReview variable after successful submission
              $userReview = getUserReviewForMovie($movieId, $userId);
               // Re-fetch comments to include the new/updated one immediately
               $comments = getMovieReviews($movieId);
          } else {
              $_SESSION['error_message'] = 'Failed to submit your review.';
          }
     }

    // Redirect using GET to prevent form resubmission on refresh
    // This also clears POST data and allows session messages to show
    header("Location: movie-details.php?id={$movieId}");
    exit;
}


// Handle Favorite/Unfavorite action (using POST for robustness)
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['toggle_favorite'])) {
     // Check if logged in to favorite
    if (!$userId) {
         $_SESSION['error_message'] = 'You must be logged in to favorite movies.';
          // Store intended URL before redirecting to login
         $_SESSION['intended_url'] = $_SERVER['REQUEST_URI'];
         header('Location: ../autentikasi/form-login.php');
         exit;
    }

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
     // Redirect back to the movie details page using GET
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

// Determine trailer source URL/Path
$trailerSrc = null; // This will hold the src for the iframe or video tag
$isYouTube = false;

if (!empty($movie['trailer_url'])) {
    // Assume YouTube URL and extract video ID
    parse_str( parse_url( $movie['trailer_url'], PHP_URL_QUERY ), $vars );
    $youtubeVideoId = $vars['v'] ?? null;
    if ($youtubeVideoId) {
        $trailerSrc = "https://www.youtube.com/embed/{$youtubeVideoId}";
        $isYouTube = true;
    } else {
         // Handle other video URL types if needed (basic passthrough)
         // Note: Directly embedding external URLs that aren't standard embed formats (like iframe src) might not work.
         // This requires testing or validation of the URL format.
         // For simplicity, if it's not a standard YouTube URL, assume it might be a direct video URL.
         $trailerSrc = htmlspecialchars($movie['trailer_url']); // Pass the URL as is
         $isYouTube = false; // Treat as potential direct video link
    }

} elseif (!empty($movie['trailer_file'])) {
    // Assume local file path, construct web accessible URL
    $trailerSrc = htmlspecialchars(WEB_UPLOAD_DIR_TRAILERS . $movie['trailer_file']); // Adjust path if necessary
    $isYouTube = false; // It's a local file
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
             position: relative; /* Needed for ::before */
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
             display: block; /* Remove extra space below image */
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
             line-height: 1.5; /* Improve vertical alignment */
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
             text-decoration: none; /* Ensure links styled as buttons don't have underline */
             color: inherit; /* Inherit color for links acting as buttons */
        }

        .action-button.watch-trailer { /* Use class for clarity */
            background-color: #e50914;
            ```css
            color: white;
        }

        .action-button.add-favorite { /* Use class for clarity */
            background-color: #333;
            color: white;
        }
         .action-button.add-favorite.favorited {
             background-color: #00ffff;
             color: #1a1a1a;
         }


        .action-button:hover {
            transform: translateY(-3px);
            opacity: 0.9;
        }
         .action-button:disabled {
             background-color: #555;
             cursor: not-allowed;
             opacity: 0.7;
             transform: none;
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
         .comment-submit-btn:disabled {
             background-color: #555;
             cursor: not-allowed;
             opacity: 0.7;
             transform: none;
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

        /* Modal styles (Trailer) */
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
                 <?php if (isAuthenticated()): // Check if logged in ?>
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favourites</span></a></li>
                 <?php endif; ?>
                <li class="active"><a href="index.php"><i class="fas fa-star"></i> <span>Review</span></a></li>
                 <?php if (isAuthenticated()): // Check if logged in ?>
                <li><a href="../manage/indeks.php"><i class="fas fa-film"></i> <span>Manage</span></a></li>
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li>
                 <?php endif; ?>
            </ul>
            <div class="bottom-links">
                <ul>
                    <?php if (isAuthenticated()): // Check if logged in ?>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li>
                    <?php else: ?>
                    <li><a href="../autentikasi/form-login.php"><i class="fas fa-sign-in-alt"></i> <span>Login</span></a></li>
                    <?php endif; ?>
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
                            <?php if ($trailerSrc): ?>
                                <button class="action-button watch-trailer" onclick="playTrailer('<?php echo $trailerSrc; ?>', <?php echo $isYouTube ? 'true' : 'false'; ?>)">
                                    <i class="fas fa-play"></i>
                                    <span>Watch Trailer</span>
                                </button>
                            <?php endif; ?>

                             <!-- Favorite/Unfavorite button (using POST form) -->
                             <?php if (isAuthenticated()): // Only show if logged in ?>
                             <form action="movie-details.php?id=<?php echo $movie['movie_id']; ?>" method="POST" style="margin:0; padding:0;">
                                 <input type="hidden" name="movie_id" value="<?php echo $movie['movie_id']; ?>">
                                 <button type="submit" name="toggle_favorite" value="<?php echo $isFavorited ? 'unfavorite' : 'favorite'; ?>"
                                         class="action-button add-favorite <?php echo $isFavorited ? 'favorited' : ''; ?>">
                                     <i class="fas fa-heart"></i>
                                     <span id="favorite-text"><?php echo $isFavorited ? 'Remove from Favorites' : 'Add to Favorites'; ?></span>
                                 </button>
                             </form>
                             <?php else: ?>
                                <!-- If not logged in, show a disabled or link button -->
                                <a href="../autentikasi/form-login.php?intended_url=<?php echo urlencode($_SERVER['REQUEST_URI']); ?>" class="action-button add-favorite" title="Login to Add to Favorites">
                                     <i class="fas fa-heart"></i>
                                     <span>Login to Favorite</span>
                                </a>
                             <?php endif; ?>
                        </div>
                    </div>
                </div>
                <div class="comments-section">
                    <h2 class="comments-header">Comments & Reviews</h2>

                     <?php if (isAuthenticated()): // Only show review input if logged in ?>
                     <div class="comment-input-area">
                         <h3><?php echo $userReview ? 'Edit Your Review' : 'Leave a Review'; ?></h3>
                         <form action="movie-details.php?id=<?php echo $movie['movie_id']; ?>" method="POST">
                             <div class="rating-input-stars" id="rating-input-stars">
                                 <i class="far fa-star" data-rating="1"></i>
                                 <i class="far fa-star" data-rating="2"></i>
                                 <i class="far fa-star" data-rating="3"></i>
                                 <i class="far fa-star" data-rating="4"></i>
                                 <i class="far fa-star" data-rating="5"></i>
                                 <!-- Set initial value from $userReview if exists -->
                                 <input type="hidden" name="rating" id="user-rating" value="<?php echo htmlspecialchars($userReview['rating'] ?? 0); ?>">
                             </div>
                             <textarea class="comment-input" name="comment" placeholder="Write your comment or review here..."><?php echo htmlspecialchars($userReview['comment'] ?? ''); ?></textarea>
                              <input type="hidden" name="submit_review" value="1"> <!-- Hidden input to identify review submission -->
                             <button type="submit" class="comment-submit-btn">Submit Review</button>
                         </form>
                     </div>
                      <?php else: ?>
                          <!-- Show login prompt if not logged in -->
                          <div class="empty-state" style="background-color: #1a1a1a; padding: 20px; border-radius: 10px; margin-bottom: 20px;">
                              <i class="fas fa-sign-in-alt" style="color: #00ffff;"></i>
                              <p>You must be <a href="../autentikasi/form-login.php?intended_url=<?php echo urlencode($_SERVER['REQUEST_URI']); ?>">logged in</a> to leave a review.</p>
                          </div>
                      <?php endif; ?>


                    <div class="comment-list">
                        <?php if (!empty($comments)): ?>
                            <?php foreach ($comments as $comment): ?>
                                <div class="comment">
                                    <div class="comment-header">
                                         <div class="user-info">
                                             <!-- Use full_name if available, fallback to username -->
                                             <img src="<?php echo htmlspecialchars($comment['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($comment['full_name'] ?? $comment['username']) . '&background=random&color=fff&size=25'); ?>" alt="Avatar" style="width: 25px; height: 25px; border-radius: 50%; margin-right: 10px; object-fit: cover;">
                                             <strong><?php echo htmlspecialchars($comment['full_name'] ?? $comment['username']); ?></strong>
                                         </div>
                                         <div style="display: flex; align-items: center; gap: 15px;">
                                            <div class="comment-rating-display">
                                                 <div class="stars">
                                                     <?php
                                                     $comment_rating = floatval($comment['rating']);
                                                     $full_stars = floor($comment_rating);
                                                     $half_star = ($comment_rating - $full_stars) >= 0.5;
                                                     $empty_stars = 5 - $full_stars - ($half_star ? 1 : 0);

                                                     for ($i = 0; $i < $full_stars; $i++) echo '<i class="fas fa-star"></i>';
                                                     if ($half_star) echo '<i class="fas fa-star-half-alt"></i>';
                                                     for ($i = 0; $i < $empty_stars; $i++) echo '<i class="far fa-star"></i>';
                                                     ?>
                                                 </div>
                                                <span>(<?php echo htmlspecialchars(number_format($comment_rating, 1)); ?>/5)</span>
                                            </div>
                                            <div class="comment-actions">
                                                <!-- Basic Placeholder Actions (Like/Dislike/Reply) -->
                                                <!-- Add logic here if you want to implement these features -->
                                                <!-- <i class="fas fa-thumbs-up" title="Like"></i> -->
                                                <!-- <i class="fas fa-thumbs-down" title="Dislike"></i> -->
                                                <!-- <i class="fas fa-reply" title="Reply"></i> -->
                                                 <span style="font-size: 0.9em; color: #888;"><?php echo (new DateTime($comment['created_at']))->format('Y-m-d H:i'); ?></span>
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
                 <!-- These elements will be dynamically updated by playTrailer JS function -->
                 <iframe id="trailer-iframe" src="" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
                 <video id="trailer-video" src="" controls></video>
            </div>
        </div>
    </div>

    <script>
    document.addEventListener('DOMContentLoaded', function() {
         // JavaScript for rating input
        const ratingStars = document.querySelectorAll('#rating-input-stars i');
        const userRatingInput = document.getElementById('user-rating');
        // Get initial rating from hidden input if user has a review
        let currentRating = parseFloat(userRatingInput.value) || 0;


         // Add data-rating attribute to stars if not already present
         ratingStars.forEach((star, index) => {
             star.setAttribute('data-rating', (index + 1).toString()); // Ensure data-rating is set as string
         });


        ratingStars.forEach(star => {
            star.addEventListener('mouseover', function() {
                const hoverRating = parseFloat(this.getAttribute('data-rating'));
                highlightStars(hoverRating, false); // Highlight based on hover
            });

            star.addEventListener('mouseout', function() {
                 // Revert to the currently selected rating
                highlightStars(currentRating, true); // Highlight based on clicked/saved state
            });

            star.addEventListener('click', function() {
                const clickedRating = parseFloat(this.getAttribute('data-rating'));
                if (currentRating === clickedRating) {
                    // If clicking the currently selected star, reset rating
                    currentRating = 0;
                } else {
                    currentRating = clickedRating; // Update selected rating
                }
                userRatingInput.value = currentRating; // Set hidden input value
                highlightStars(currentRating, true); // Highlight and mark as rated
            });
        });

        function highlightStars(rating, isClickedState) {
            ratingStars.forEach((star, index) => {
                const starRating = index + 1; // Use index + 1 for star value (1 to 5)

                star.classList.remove('hovered', 'rated', 'fas', 'far', 'fa-star-half-alt'); // Reset classes

                if (starRating <= rating) {
                    star.classList.add('fas', 'fa-star'); // Full star
                    star.classList.add(isClickedState ? 'rated' : 'hovered');
                } else if (starRating - 0.5 <= rating) {
                     star.classList.add('fas', 'fa-star-half-alt'); // Half star
                     star.classList.add(isClickedState ? 'rated' : 'hovered');
                } else {
                    star.classList.add('far', 'fa-star'); // Empty star
                }
            });
        }

        // Initial highlight based on the loaded user review rating
        highlightStars(currentRating, true);


         // JavaScript for trailer modal
        const trailerModal = document.getElementById('trailer-modal');
        const trailerIframe = document.getElementById('trailer-iframe'); // For YouTube
        const trailerVideo = document.getElementById('trailer-video'); // For local files
        const watchTrailerButton = document.querySelector('.action-button.watch-trailer');

        function playTrailer(videoSrc, isYouTube = true) {
            if (videoSrc) {
                 // Hide both initially
                 trailerIframe.style.display = 'none';
                 trailerVideo.style.display = 'none';
                 // Ensure video is paused and iframe src is cleared before setting the new source
                 trailerVideo.pause();
                 trailerVideo.removeAttribute('src'); // Remove src to unload
                 trailerIframe.removeAttribute('src'); // Remove src

                 if (isYouTube) {
                    trailerIframe.src = videoSrc + '?autoplay=1'; // Add autoplay
                    trailerIframe.style.display = 'block';
                 } else {
                    trailerVideo.src = videoSrc;
                    trailerVideo.style.display = 'block';
                    trailerVideo.load(); // Load the video
                    trailerVideo.play(); // Start playing
                 }
                trailerModal.classList.add('active');
            } else {
                // This case should ideally be prevented by the PHP if check,
                // but keep as a fallback.
                alert('Trailer not available.');
            }
        }

        function closeTrailer() {
            // Stop playback and clear sources when closing
            if (trailerIframe && trailerIframe.style.display !== 'none') {
                 trailerIframe.contentWindow.postMessage('{"event":"command","func":"stopVideo","args":""}', '*'); // YouTube stop
                 trailerIframe.removeAttribute('src'); // Clear src completely
                 trailerIframe.style.display = 'none';
            }
            if (trailerVideo && trailerVideo.style.display !== 'none') {
                 trailerVideo.pause(); // Pause local video
                 trailerVideo.currentTime = 0; // Reset time
                 trailerVideo.removeAttribute('src'); // Clear src completely
                 trailerVideo.style.display = 'none';
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

        // Make the Watch Trailer button trigger the modal
        if(watchTrailerButton) {
             // Get trailer src and type from PHP variables embedded in attributes or data
             const trailerSrc = watchTrailerButton.getAttribute('onclick').match(/playTrailer\('(.*?)', (true|false)\)/);
             if (trailerSrc && trailerSrc[1]) {
                  const src = trailerSrc[1];
                  const isYouTube = trailerSrc[2] === 'true';

                  // Remove the inline onclick
                  watchTrailerButton.removeAttribute('onclick');

                  // Add the event listener
                  watchTrailerButton.addEventListener('click', function() {
                      playTrailer(src, isYouTube);
                  });
             }
        }


     // Helper function for HTML escaping (client-side) - already exists, ensuring it's here
     function htmlspecialchars(str) {
         if (typeof str !== 'string') return str;
          // Create a temporary DOM element to leverage browser's escaping
         const div = document.createElement('div');
         div.appendChild(document.createTextNode(str));
         return div.innerHTML;
     }
    }); // End DOMContentLoaded
    </script>
</body>
</html>
```

---

**`review/styles.css`** (No change needed)

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
     position: relative; /* Needed for ::before */
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
     display: block; /* Remove extra space below image */
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
    line-height: 1.5; /* Improve vertical alignment */
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

**`database`** (The schema dump you provided)

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
    password VARCHAR(255), -- Password can be NULL for Google users
    age INT,
    gender ENUM('Laki-laki', 'Perempuan'),
    bio TEXT, -- Added bio column
    profile_image VARCHAR(255),
    google_id VARCHAR(255) UNIQUE, -- Added google_id column
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
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    UNIQUE KEY unique_user_movie_review (movie_id, user_id) -- Added unique key here
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
CREATE INDEX idx_users_google_id ON users(google_id); -- Added index for Google ID
```

---

### Step 4: Get Google API Key and Configuration

To make Google Sign-In work, you need a Google Cloud Project and OAuth 2.0 credentials.

1.  Go to the [Google Cloud Console](https://console.cloud.google.com/).
2.  If you don't have a project, create a new one.
3.  From the navigation menu, go to **APIs & Services** > **Credentials**.
4.  Click **+ CREATE CREDENTIALS** and select **OAuth client ID**.
5.  If prompted, configure the OAuth consent screen first (you only need to do this once per project).
    *   Select "External" user type.
    *   Fill in required fields like App name (e.g., "RATE-TALES"), User support email, and Developer contact information.
    *   Add scopes if needed (basic profile/email are usually sufficient).
    *   Add test users if you are not publishing the app.
    *   Go back to Credentials.
6.  Select **Web application** as the Application type.
7.  Give your OAuth client a name (e.g., "Rate Tales Web Client").
8.  Under **Authorized JavaScript origins**, add the origin(s) where your application is hosted.
    *   For local development, this is usually `http://localhost` or `http://127.0.0.1`.
    *   If you access it via a subfolder like `http://localhost/R-TALES_EX-C1`, you might still only need the base origin `http://localhost`. Test this.
    *   **Important:** Do NOT include trailing slashes or paths (`/R-TALES_EX-C1/`) here, just the protocol and domain/port. E.g., `http://localhost`, `https://yourdomain.com`.
9.  Under **Authorized redirect URIs**, you **don't need to add anything for the Google Identity Services (GSI) JavaScript library** method we're using. The verification happens via AJAX.
10. Click **Create**.
11. A modal will appear with your **Client ID**. Copy this ID.
12. Paste this ID into your `includes/config.php` file, replacing `'YOUR_GOOGLE_CLIENT_ID'`:

    ```php
    define('GOOGLE_CLIENT_ID', 'PASTE_YOUR_CLIENT_ID_HERE');
    ```

Now, when you load the login or register pages, the Google Sign-In button should appear, and clicking it should initiate the Google login flow and send the token to your `verify_google_login.php` script for authentication/registration.

---

### Summary of Fixes and Enhancements:

1.  **Database:** Added `google_id` column to `users` and `UNIQUE KEY (movie_id, user_id)` to `reviews`.
2.  **Login/Logout:** Refined session handling, error/success message display, and input persistence on redirect. Logout now specifically unsets user session variables and the google login marker.
3.  **Google Sign-In:** Integrated Google Identity Services client-side and created `verify_google_login.php` for server-side token verification, supporting login for existing Google users, linking Google ID to existing email users, and registering new users via Google.
4.  **Trailer Playback:** Modified `review/movie-details.php` and its JavaScript to correctly handle both YouTube URLs (using iframe) and local video files (using `<video>` tag).
5.  **Manage Edit:** Created the `manage/edit.php` file and implemented logic to fetch existing movie data, pre-fill the form, handle updates (including file uploads and genre changes), and delete old files on successful updates.
6.  **Profile (acc_page):** Fixed bio edit logic to handle saving and display correctly. Updated profile image upload script (`upload_profile_image.php`) to delete the old image file after a successful update and added some error handling for file uploads. Made uploaded movie cards clickable. Added client-side HTML escaping helper for search/display logic.
7.  **General:** Improved error/success message display consistency across pages. Added HTML escaping for outputting user-provided data where appropriate. Added `isAuthenticated()` check in `beranda/index.php` sidebar.

---

**To make everything work:**

1.  **Apply the database schema changes** using the SQL commands provided.
2.  **Replace the placeholder `YOUR_GOOGLE_CLIENT_ID`** in `includes/config.php` with your actual Google Client ID.
3.  **Place the new and updated files** in their correct directories.
4.  Ensure your `uploads/posters/`, `uploads/trailers/`, and `uploads/profile_images/` directories exist and are writable by your web server process. You might need to set permissions (e.g., `chmod 775 uploads/ uploads/posters/ uploads/trailers/ uploads/profile_images/`).
5.  Make sure the path `/R-TALES_EX-C1/` used in `config/config.php` (for `redirectIfNotAuthenticated` and Google Login redirect) matches your actual project structure. If your project is hosted directly at the root (e.g., `http://localhost/`), you might need to adjust this path, or preferably, use absolute URLs starting from the web root if you have defined one. (I've kept the original path logic based on the provided code).

Test each feature: Login (local and Google), Registration (local and Google), Logout, View Home, View Reviews, View Movie Details (check trailer, favorite button), Manage (view uploaded, upload new, edit existing, delete).

This comprehensive update should address all the issues you listed and integrate the Google Sign-In feature smoothly. Let me know if you encounter any further problems!
