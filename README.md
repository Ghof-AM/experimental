Oke, mari kita susun ulang dan integrasikan kode-kode tersebut menjadi satu aplikasi web yang berfungsi, dengan CSS digabung menjadi satu file dan struktur folder yang lebih terorganisir.

**Struktur Folder yang Disarankan:**

```
ratingtales/
├── acc_page/
│   └── index.php
├── autentikasi/
│   ├── form-login.php
│   ├── form-register.php
│   ├── generate_captcha.php
│   └── logout.php
├── beranda/
│   └── index.php
├── config/
│   └── database.php
├── css/
│   └── style.css      <-- Semua CSS digabung di sini
├── favorite/
│   └── index.php
├── gambar/              <-- Folder untuk gambar latar/aset umum
│   └── 5302920.jpg      <-- Gambar latar autentikasi
│   └── untitled142_20250310223718.png <-- Icon (disesuaikan path-nya)
├── includes/            <-- Folder untuk helper functions, mungkin nanti header/footer
│   └── auth_helper.php  <-- Fungsi cek login, dll.
├── js/                  <-- Folder untuk file JS
│   ├── animation.js     <-- Animasi form autentikasi
│   └── main.js          <-- JS umum, slider beranda, popup review, dll.
├── manage/
│   ├── index.php
│   └── upload.php
├── review/
│   ├── index.php
│   └── movie-details.php
├── uploads/             <-- Folder untuk file yang diunggah (poster, trailer, profil)
│   ├── posters/
│   ├── profiles/
│   └── trailers/
└── index.php            <-- Halaman utama (mungkin redirect)
```

**Perubahan Utama yang Akan Dilakukan:**

1.  **Satu File CSS:** Semua gaya CSS dari file `styles.css` di setiap folder dan `upload.css` akan digabungkan ke `css/style.css`. Inline style di `review/movie-details.php` juga dipindahkan ke sini. Path gambar latar di CSS akan diperbarui (`url(../gambar/...)`). Skema warna akan diselaraskan ke yang digunakan di banyak tempat (`#00ffff`).
2.  **Integrasi Database:** Data hardcoded di semua halaman (kecuali autentikasi awal) akan diganti dengan data yang diambil dari database menggunakan fungsi di `config/database.php`.
3.  **Fungsionalitas Penuh:**
    *   **Autentikasi:** Login, Register, Logout akan berfungsi penuh menggunakan database dan session, termasuk CAPTCHA server-side.
    *   **Beranda:** Menampilkan film dari database. Slider dan grid film berfungsi.
    *   **Review:** Menampilkan film dari database. Popup detail film berfungsi dengan data dinamis dari database, termasuk rating interaktif dan menampilkan komentar (komentar akan hardcode sementara karena form submit belum penuh). Link "Read More" akan mengarah ke halaman detail film dengan ID film.
    *   **Movie Details:** Mengambil detail film dan komentar terkait berdasarkan ID film dari URL. Menampilkan data dinamis, termasuk poster besar dan deskripsi penuh. Tombol "Watch Trailer" akan memutar trailer (asumsi URL YouTube atau file). Tombol "Add to Favorites" akan diimplementasikan (perlu AJAX). Komentar akan ditampilkan (masih hardcode, tapi struktur siap).
    *   **Favorites:** Menampilkan film yang ditandai favorit oleh pengguna yang login, diambil dari database. Fungsi "Remove from Favorites" diimplementasikan (perlu AJAX). Search JS tetap klien-side.
    *   **Manage:** Menampilkan film yang diunggah oleh pengguna yang login. Fungsi upload film akan berfungsi (memproses form POST, menyimpan file, menyimpan ke database).
    *   **Profile (`acc_page`):** Menampilkan data pengguna yang login, diambil dari database. Bagian "My Post" dihapus karena tidak ada tabel posts. Fungsi edit nama, username, bio akan diimplementasikan (perlu AJAX dan endpoint server-side).
4.  **AJAX:** Beberapa fitur (update profil, add/remove favorite, submit review/comment) memerlukan endpoint PHP terpisah dan logika AJAX di sisi klien. Ini akan ditambahkan.
5.  **Helper Functions:** Fungsi seperti cek status login (`require_login()`) akan ditambahkan di `includes/auth_helper.php`.
6.  **Struktur Kode PHP:** Menambahkan `require_once` untuk `config/database.php` dan `includes/auth_helper.php` di awal setiap file PHP yang memerlukannya.
7.  **Penamaan:** Mengganti `manage/indeks.php` menjadi `manage/index.php` untuk konsistensi.
8.  **File Upload Security:** Implementasi upload file akan mencakup pemeriksaan dasar tipe file dan pembuatan nama file unik, namun perlu diingat bahwa keamanan upload file yang sesungguhnya membutuhkan validasi lebih lanjut (misalnya, menggunakan library untuk memeriksa isi file) dan pengaturan server yang tepat.

**File-file yang Akan Dibuat/Diubah:**

1.  `css/style.css` (BARU & GABUNGAN)
2.  `config/database.php` (MODIFIKASI: tambah helper, fungsi rating, update user)
3.  `includes/auth_helper.php` (BARU)
4.  `js/animation.js` (PINDAH)
5.  `js/main.js` (BARU & GABUNGAN JS LAIN)
6.  `index.php` (BARU)
7.  `autentikasi/form-login.php` (MODIFIKASI)
8.  `autentikasi/form-register.php` (MODIFIKASI)
9.  `autentikasi/generate_captcha.php` (PINDAH & MODIFIKASI)
10. `autentikasi/logout.php` (PINDAH & MODIFIKASI)
11. `beranda/index.php` (MODIFIKASI)
12. `favorite/index.php` (MODIFIKASI)
13. `manage/index.php` (MODIFIKASI)
14. `manage/upload.php` (MODIFIKASI)
15. `acc_page/index.php` (MODIFIKASI)
16. `review/index.php` (MODIFIKASI)
17. `review/movie-details.php` (MODIFIKASI)
18. `acc_page/update_profile.php` (BARU - AJAX endpoint)
19. `favorite/toggle_favorite.php` (BARU - AJAX endpoint)
20. `review/submit_review.php` (BARU - AJAX endpoint)
21. `manage/upload_handler.php` (BARU - Form handler)

**Catatan Penting:**

*   **User ID:** Logika aplikasi akan sangat bergantung pada `$_SESSION['user_id']` untuk menentukan pengguna yang sedang login. Pastikan semua halaman yang memerlukan data pengguna atau mengelola data pengguna melindunginya dengan cek login.
*   **File Upload Paths:** Path untuk menyimpan file yang diunggah (`uploads/`) harus bisa diakses oleh server PHP (izin tulis). Path di database akan menjadi relatif dari root web atau path absolut yang bisa diakses browser. Kita akan gunakan path relatif `/uploads/...`.
*   **Error Handling:** Implementasi akan mencakup penanganan error dasar, tetapi aplikasi produksi memerlukan logging error yang lebih kuat dan pesan kesalahan yang lebih ramah pengguna.
*   **Database:** Asumsi tabel sudah dibuat sesuai skema yang diberikan.

Mari kita mulai dengan membuat file-file tersebut.

---

**1. `css/style.css` (Gabungan Semua CSS)**

Buat file `css/style.css` dan tempelkan semua isi dari `acc_page/styles.css`, `beranda/styles.css`, `favorite/styles.css`, `manage/styles.css`, `manage/upload.css`, `review/styles.css`, `autentikasi/style.css` ke dalamnya. Kemudian lakukan penyesuaian:

*   Ubah semua `url(...)` yang merujuk ke gambar latar autentikasi agar path-nya benar dari `css/style.css`, yaitu `url(../gambar/5302920.jpg)`.
*   Standardisasi warna utama dari `#00e4f9` (autentikasi) menjadi `#00ffff` (lainnya). Ganti semua `#00e4f9` dengan `#00ffff` kecuali jika ada alasan khusus (misalnya, untuk variasi hover/focus, gunakan `#00cccc` atau `#00e4f9` jika suka). Kita akan pakai `#00ffff` sebagai warna utama, `#00cccc` untuk hover/focus.
*   Perbaiki nama kelas atau ID jika ada konflik (misalnya, beberapa file memiliki `.container` atau `.sidebar`). Pastikan selector CSS cukup spesifik (misalnya, `.manage-content` vs `.review-content` jika ada gaya berbeda). Namun, struktur HTML yang diberikan sudah cukup membedakan section utama, jadi `.main-content` umum bisa dipakai.
*   Pindahkan inline style dari `review/movie-details.php` ke sini dengan selector yang sesuai.

*(Karena file CSS gabungan sangat panjang, saya tidak akan menempelkannya di sini. Anda harus melakukannya secara manual dengan menggabungkan semua file CSS asli dan style inline, lalu melakukan penyesuaian path gambar dan warna seperti dijelaskan di atas).*

**2. `config/database.php` (Modifikasi)**

Tambahkan fungsi `generateRandomString`, `getMoviesByUserId`, `updateUserField`, `getAverageMovieRating`, dan `isFavorite`.

```php
<?php
// config/database.php

if (session_status() == PHP_SESSION_NONE) {
    session_start();
}

$host = 'localhost';
$dbname = 'ratingtales';
$username = 'root';
$password = '';

try {
    $pdo = new PDO("mysql:host=$host;dbname=$dbname", $username, $password);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    $pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
} catch(PDOException $e) {
    // In production, log this error and show a generic message
    error_log("Database Connection failed: " . $e->getMessage());
    die("Sistem sedang dalam perbaikan. Silakan coba lagi nanti."); // Generic user-facing error
}

// --- Helper Functions ---

// Function to generate random string for CAPTCHA
function generateRandomString($length = 6) {
    $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    $charactersLength = strlen($characters);
    $randomString = '';
    for ($i = 0; $i < $length; $i++) {
        $randomString .= $characters[rand(0, $charactersLength - 1)];
    }
    return $randomString;
}

// --- User Functions ---

function createUser($full_name, $username, $email, $password, $age, $gender, $profile_image = null) {
    global $pdo;
    $hashedPassword = password_hash($password, PASSWORD_DEFAULT);
    $stmt = $pdo->prepare("INSERT INTO users (full_name, username, email, password, age, gender, profile_image) VALUES (?, ?, ?, ?, ?, ?, ?)");
    return $stmt->execute([$full_name, $username, $email, $hashedPassword, $age, $gender, $profile_image]);
}

function getUserById($userId) {
    global $pdo;
    $stmt = $pdo->prepare("SELECT user_id, full_name, username, email, profile_image, bio, created_at FROM users WHERE user_id = ?");
    $stmt->execute([$userId]);
    return $stmt->fetch();
}

function getUserByUsernameOrEmail($usernameOrEmail) {
     global $pdo;
     $stmt = $pdo->prepare("SELECT user_id, password FROM users WHERE username = ? OR email = ?");
     $stmt->execute([$usernameOrEmail, $usernameOrEmail]);
     return $stmt->fetch();
}


function updateUserField($userId, $field, $value) {
    global $pdo;
    // Basic validation for allowed fields to prevent SQL injection via field name
    $allowedFields = ['full_name', 'username', 'email', 'bio', 'profile_image'];
    if (!in_array($field, $allowedFields)) {
        error_log("Attempted to update disallowed user field: " . $field);
        return false;
    }

    // Add specific validation/logic for username if needed (e.g., uniqueness check)
    if ($field === 'username') {
        $stmtCheck = $pdo->prepare("SELECT COUNT(*) FROM users WHERE username = ? AND user_id <> ?");
        $stmtCheck->execute([$value, $userId]);
        if ($stmtCheck->fetchColumn() > 0) {
             // Username already exists for another user
            return false; // Indicate failure due to uniqueness constraint
        }
    }
     // Add specific validation/logic for email if needed

    $sql = "UPDATE users SET " . $field . " = ? WHERE user_id = ?";
    $stmt = $pdo->prepare($sql);
    return $stmt->execute([$value, $userId]);
}


// --- Movie Functions ---

function createMovie($title, $summary, $release_date, $duration_hours, $duration_minutes, $age_rating, $poster_image, $trailer_url, $trailer_file, $uploaded_by) {
    global $pdo;
    $stmt = $pdo->prepare("INSERT INTO movies (title, summary, release_date, duration_hours, duration_minutes, age_rating, poster_image, trailer_url, trailer_file, uploaded_by) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)");
    $stmt->execute([$title, $summary, $release_date, $duration_hours, $duration_minutes, $age_rating, $poster_image, $trailer_url, $trailer_file, $uploaded_by]);
    return $pdo->lastInsertId(); // Return the ID of the newly inserted movie
}

function addMovieGenre($movie_id, $genre) {
    global $pdo;
    // Validate genre against ENUM values if necessary
    $stmt = $pdo->prepare("INSERT INTO movie_genres (movie_id, genre) VALUES (?, ?)");
    // Use try-catch for potential duplicate entry errors if genre already exists for movie
    try {
        return $stmt->execute([$movie_id, $genre]);
    } catch (PDOException $e) {
        // Log or handle duplicate entry if needed, for now just return false
        if ($e->getCode() == 23000) { // SQLSTATE 23000 is for integrity constraint violation
             // Duplicate entry, genre already added for this movie
             return false;
        }
        // Rethrow or handle other PDO errors
        throw $e;
    }
}

function getMovieById($movieId) {
    global $pdo;
    // Use LEFT JOIN to ensure movies with no genres are still returned
    $stmt = $pdo->prepare("SELECT m.*, GROUP_CONCAT(mg.genre) as genres FROM movies m LEFT JOIN movie_genres mg ON m.movie_id = mg.movie_id WHERE m.movie_id = ? GROUP BY m.movie_id");
    $stmt->execute([$movieId]);
    $movie = $stmt->fetch();
    if ($movie && $movie['genres'] !== null) {
        $movie['genres'] = explode(',', $movie['genres']);
    } else if ($movie) {
         $movie['genres'] = []; // Ensure genres is an array even if null
    }
    return $movie;
}

function getAllMovies() {
    global $pdo;
     // Use LEFT JOIN to ensure movies with no genres are still returned
    $stmt = $pdo->prepare("SELECT m.*, GROUP_CONCAT(mg.genre) as genres FROM movies m LEFT JOIN movie_genres mg ON m.movie_id = mg.movie_id GROUP BY m.movie_id ORDER BY m.created_at DESC");
    $stmt->execute();
    $movies = $stmt->fetchAll();
    // Process genres for each movie
    foreach ($movies as &$movie) {
        if ($movie['genres'] !== null) {
            $movie['genres'] = explode(',', $movie['genres']);
        } else {
            $movie['genres'] = [];
        }
    }
    return $movies;
}

function getMoviesByUserId($userId) {
    global $pdo;
    // Use LEFT JOIN to ensure movies with no genres are still returned
    $stmt = $pdo->prepare("SELECT m.*, GROUP_CONCAT(mg.genre) as genres FROM movies m LEFT JOIN movie_genres mg ON m.movie_id = mg.movie_id WHERE m.uploaded_by = ? GROUP BY m.movie_id ORDER BY m.created_at DESC");
    $stmt->execute([$userId]);
    $movies = $stmt->fetchAll();
     // Process genres for each movie
    foreach ($movies as &$movie) {
        if ($movie['genres'] !== null) {
            $movie['genres'] = explode(',', $movie['genres']);
        } else {
            $movie['genres'] = [];
        }
    }
    return $movies;
}

function getAverageMovieRating($movieId) {
    global $pdo;
    $stmt = $pdo->prepare("SELECT AVG(rating) as average_rating FROM reviews WHERE movie_id = ?");
    $stmt->execute([$movieId]);
    $result = $stmt->fetch();
    // Return average rating, formatted to 1 decimal place, or 0.0 if no reviews
    return $result['average_rating'] !== null ? number_format($result['average_rating'], 1) : '0.0';
}


// --- Review Functions ---

function createReview($movie_id, $user_id, $rating, $comment) {
    global $pdo;
    // Basic validation on rating if necessary (e.g., 1.0 to 5.0)
    $stmt = $pdo->prepare("INSERT INTO reviews (movie_id, user_id, rating, comment) VALUES (?, ?, ?, ?)");
    // Use try-catch for potential errors
    try {
         return $stmt->execute([$movie_id, $user_id, $rating, $comment]);
    } catch (PDOException $e) {
         error_log("Error creating review: " . $e->getMessage());
         return false; // Indicate failure
    }
}

function getMovieReviews($movieId) {
    global $pdo;
    $stmt = $pdo->prepare("SELECT r.*, u.username, u.profile_image FROM reviews r JOIN users u ON r.user_id = u.user_id WHERE r.movie_id = ? ORDER BY r.created_at DESC");
    $stmt->execute([$movieId]);
    return $stmt->fetchAll();
}

// --- Favorite Functions ---

function addToFavorites($movie_id, $user_id) {
    global $pdo;
    // Prevent duplicate favorites using UNIQUE constraint on (movie_id, user_id)
    $stmt = $pdo->prepare("INSERT INTO favorites (movie_id, user_id) VALUES (?, ?)");
    try {
        return $stmt->execute([$movie_id, $user_id]);
    } catch (PDOException $e) {
        // Check if it's a duplicate entry error (SQLSTATE 23000)
        if ($e->getCode() == 23000) {
             // Already a favorite, not an error, maybe return true or specific code
             return true; // Treat as success if already exists
        }
        // Log or handle other errors
        error_log("Error adding favorite: " . $e->getMessage());
        return false;
    }
}

function removeFromFavorites($movie_id, $user_id) {
    global $pdo;
    $stmt = $pdo->prepare("DELETE FROM favorites WHERE movie_id = ? AND user_id = ?");
    return $stmt->execute([$movie_id, $user_id]);
}

function getUserFavorites($userId) {
    global $pdo;
     // Use LEFT JOIN for genres
    $stmt = $pdo->prepare("SELECT m.*, GROUP_CONCAT(mg.genre) as genres FROM favorites f JOIN movies m ON f.movie_id = m.movie_id LEFT JOIN movie_genres mg ON m.movie_id = mg.movie_id WHERE f.user_id = ? GROUP BY m.movie_id ORDER BY f.created_at DESC");
    $stmt->execute([$userId]);
    $movies = $stmt->fetchAll();
     // Process genres for each movie
    foreach ($movies as &$movie) {
        if ($movie['genres'] !== null) {
            $movie['genres'] = explode(',', $movie['genres']);
        } else {
            $movie['genres'] = [];
        }
    }
     // Optionally fetch average rating for favorites list - requires looping or complex query
     // For simplicity, let's fetch average rating for each favorite after getting the list
    foreach ($movies as &$movie) {
         $movie['average_rating'] = getAverageMovieRating($movie['movie_id']);
    }
    return $movies;
}

function isFavorite($movieId, $userId) {
    global $pdo;
    if (!$userId) return false; // Cannot be favorite if not logged in
    $stmt = $pdo->prepare("SELECT COUNT(*) FROM favorites WHERE movie_id = ? AND user_id = ?");
    $stmt->execute([$movieId, $userId]);
    return $stmt->fetchColumn() > 0;
}

?>
```

**3. `includes/auth_helper.php` (Baru)**

Buat file `includes/auth_helper.php` untuk fungsi pembantu autentikasi.

```php
<?php
// includes/auth_helper.php

// Ensure session is started (should be done in config.php, but double check)
if (session_status() == PHP_SESSION_NONE) {
    session_start();
}

// Function to check if a user is logged in
function is_logged_in() {
    return isset($_SESSION['user_id']);
}

// Function to redirect to login page if not logged in
function require_login() {
    if (!is_logged_in()) {
        // Store the requested page URL in session to redirect back after login
        $_SESSION['redirect_url'] = $_SERVER['REQUEST_URI'];
        header('Location: /ratingtales/autentikasi/form-login.php'); // Adjust path as necessary
        exit;
    }
}

// Function to get the logged-in user's ID
function get_user_id() {
    return $_SESSION['user_id'] ?? null;
}

// Function to get the logged-in user's data
function get_logged_in_user_data() {
    if (is_logged_in()) {
        require_once __DIR__ . '/../config/database.php'; // Include database functions
        return getUserById(get_user_id());
    }
    return null;
}

?>
```

**4. `js/animation.js` (Pindah)**

Pindahkan isi `autentikasi/animation.js` ke `js/animation.js`. Sesuaikan path di file HTML yang memanggilnya.

```javascript
// js/animation.js

document.addEventListener('DOMContentLoaded', () => {
    const formContainers = document.querySelectorAll('.form-container');

    setTimeout(() => {
        formContainers.forEach(container => {
            container.classList.add('show');
        });
    }, 100);
});
```

**5. `js/main.js` (Baru & Gabungan JS Lain)**

Gabungkan JS dari `beranda/index.php`, `favorite/index.php`, `review/index.php`, `review/movie-details.php`, `manage/index.php`, dan `acc_page/index.php`. Buat file `js/main.js`.

```javascript
// js/main.js

document.addEventListener('DOMContentLoaded', function() {
    // --- Global/Helper Functions ---
    // Function to update star icons based on a rating value
    function updateStarIcons(containerElement, ratingValue) {
        const stars = containerElement.querySelectorAll('i.fa-star, i.fa-star-half-alt, i.far.fa-star');
        const rating = parseFloat(ratingValue);

        stars.forEach((star, index) => {
            star.classList.remove('fas', 'far', 'fa-star-half-alt');
            const starValue = index + 1;

            if (starValue <= rating) {
                star.classList.add('fas', 'fa-star');
            } else if (starValue - 0.5 <= rating) {
                star.classList.add('fas', 'fa-star-half-alt');
            } else {
                star.classList.add('far', 'fa-star');
            }
        });
    }


    // --- Beranda (Home) Slider ---
    const slides = document.querySelectorAll('.featured-movie-slider .slide');
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

        // Initial show
        showSlide(currentSlide);
        // Change slide every 5 seconds
        setInterval(nextSlide, 5000);
    }


    // --- Search Functionality (Used on multiple pages) ---
    const searchInput = document.querySelector('.search-bar input');
    const searchButton = document.querySelector('.search-bar button'); // Assuming a search button exists

    if (searchInput) {
        // Function to perform search filtering on movie/item cards
        function performSearch() {
            const searchTerm = searchInput.value.toLowerCase();
            const grid = document.querySelector('.review-grid, .movies-grid'); // Adjust selectors as needed
            const cards = document.querySelectorAll('.movie-card'); // Adjust selector

            let hasVisibleCards = false;

            cards.forEach(card => {
                const titleElement = card.querySelector('h3'); // Adjust selector
                const infoElement = card.querySelector('.movie-info'); // For genre/year (on review/favorite)

                const title = titleElement ? titleElement.textContent.toLowerCase() : '';
                const info = infoElement ? infoElement.textContent.toLowerCase() : ''; // Combine text from info element

                if (title.includes(searchTerm) || info.includes(searchTerm)) {
                    card.style.display = '';
                    hasVisibleCards = true;
                } else {
                    card.style.display = 'none';
                }
            });

            // Handle empty state messages based on search results
            const existingEmptyState = document.querySelector('.search-empty-state');
            if (!hasVisibleCards && searchTerm !== '') {
                 if (grid) {
                     if (!existingEmptyState) {
                         const emptyState = document.createElement('div');
                         emptyState.className = 'empty-state search-empty-state';
                         // Customize message based on page (e.g., "No favorites found", "No movies found")
                         let message = 'No results found';
                         let subtitle = 'Try a different search term';
                         if (document.body.classList.contains('favorites-page')) { // Add body class to differentiate
                             message = 'No favorites found';
                         } else if (document.body.classList.contains('manage-page')) { // Add body class
                             message = 'No movies found';
                         }
                         emptyState.innerHTML = `
                             <i class="fas fa-search"></i>
                             <p>${message}</p>
                             <p class="subtitle">${subtitle}</p>
                         `;
                         grid.appendChild(emptyState);
                     }
                 }
            } else {
                if (existingEmptyState) {
                    existingEmptyState.remove();
                }
                 // Also check for the default "no movies uploaded yet" state on manage page
                 if (searchTerm === '' && document.body.classList.contains('manage-page')) {
                      const defaultEmptyState = document.querySelector('.empty-state:not(.search-empty-state)');
                      if (cards.length === 0 && !defaultEmptyState) {
                           // Re-add default empty state if no cards AND no search term
                            // This requires knowing the original HTML structure
                           // For now, just ensure search empty state is removed
                      }
                 }
            }
        }

        // Add event listeners for search
        searchInput.addEventListener('input', performSearch);
        if (searchButton) {
            searchButton.addEventListener('click', function(e) {
                e.preventDefault(); // Prevent form submission if button is in a form
                performSearch();
            });
        }

         // Handle initial state on Manage page if no movies are loaded
         if (document.body.classList.contains('manage-page')) {
              const movieCards = document.querySelectorAll('.movies-grid .movie-card');
              const defaultEmptyState = document.querySelector('.empty-state:not(.search-empty-state)');
              if (movieCards.length > 0 && defaultEmptyState) {
                   defaultEmptyState.style.display = 'none'; // Hide default if movies are loaded
              }
         }

    } // End if searchInput exists


    // --- Review Page Popup and Rating ---
    const movieDetailsPopup = document.getElementById('movie-details-popup');
    let currentMovieData = {}; // To store data for the popup/details page

    // Function to open the popup with movie data
    window.showMovieDetailsPopup = function(movieId, title, year, genre, description, rating) {
         // Instead of passing just text, ideally fetch full movie data by ID here
         // For now, we use the data passed from the card click
         currentMovieData = {
             id: movieId, // Pass the actual movie ID
             title: title,
             year: year,
             genre: genre,
             description: description, // Assuming description is already HTML escaped if needed
             rating: rating
         };

        document.getElementById('popup-title').textContent = currentMovieData.title;
        document.getElementById('popup-year-genre').textContent = currentMovieData.year + ' | ' + currentMovieData.genre;
        document.getElementById('popup-description').textContent = currentMovieData.description;
        // document.getElementById('popup-rating').textContent = currentMovieData.rating; // This is the average, maybe set dynamically

        // Update rating stars in the popup
        const popupRatingStarsContainer = document.querySelector('#movie-details-popup .rating-stars');
        if (popupRatingStarsContainer) {
             // Clear existing stars if any
             popupRatingStarsContainer.innerHTML = '';
             // Recreate 5 empty stars with data-rating
             for (let i = 1; i <= 5; i++) {
                  const star = document.createElement('i');
                  star.className = 'far fa-star'; // Start with empty stars for user to click
                  star.setAttribute('data-rating', i);
                  popupRatingStarsContainer.appendChild(star);
             }

             // Add event listeners for rating
             popupRatingStarsContainer.querySelectorAll('i').forEach(star => {
                 star.addEventListener('click', handleRatingClick);
                 star.addEventListener('mouseover', handleRatingHover);
                 star.addEventListener('mouseout', handleRatingOut);
             });

             // If showing average rating in popup, update those stars separately
             const avgRatingSpan = document.querySelector('#movie-details-popup .popup-rating span');
             if (avgRatingSpan) {
                 avgRatingSpan.textContent = `${currentMovieData.rating}/5`; // Display average rating
             }
        }


        if (movieDetailsPopup) {
            movieDetailsPopup.classList.add('active');
        }

        // Close popup when clicking outside
        if (movieDetailsPopup) {
             movieDetailsPopup.onclick = function(e) {
                 // Check if click target is the modal background itself, not the content
                 if (e.target === movieDetailsPopup) {
                     hideMovieDetailsPopup();
                 }
             };
        }
    }

    // Function to hide the popup
    window.hideMovieDetailsPopup = function() {
        if (movieDetailsPopup) {
            movieDetailsPopup.classList.remove('active');
        }
    }


    // Handle Rating Interaction in Popup (Client-side visual feedback)
    let userSelectedRating = 0; // Store user's potential rating

    function handleRatingHover() {
        const rating = this.getAttribute('data-rating');
        const stars = this.parentElement.querySelectorAll('i');
        stars.forEach((star, index) => {
            if (index < rating) {
                star.className = 'fas fa-star';
            } else {
                star.className = 'far fa-star';
            }
        });
    }

     function handleRatingOut() {
         // Reset to user's selected rating, or clear if none selected
         const stars = this.parentElement.querySelectorAll('i');
          if (userSelectedRating > 0) {
               highlightStars(stars, userSelectedRating, 'fas');
          } else {
               highlightStars(stars, 0, 'far'); // Clear all stars
          }
     }

    function handleRatingClick() {
        userSelectedRating = this.getAttribute('data-rating');
        const stars = this.parentElement.querySelectorAll('i');
        highlightStars(stars, userSelectedRating, 'fas'); // Highlight based on click
        // TODO: Send this rating to the server via AJAX using currentMovieData.id and userSelectedRating
        console.log(`User rated ${userSelectedRating} stars for movie ID: ${currentMovieData.id}`);
        // After sending, you might want to update the average rating displayed
        // Or just store the user's rating visually until page refresh
        alert(`Thank you for rating ${userSelectedRating} stars! (This rating is not saved yet)`); // Placeholder
    }

     function highlightStars(stars, rating, className) {
          stars.forEach((star, index) => {
               star.className = (index < rating) ? `fas fa-star` : `far fa-star`;
          });
     }


    // "Read More" button in Review Popup
    window.openMovieDetails = function() {
        if (currentMovieData && currentMovieData.id) {
            // Redirect to movie-details page with movie ID in URL
            window.location.href = `movie-details.php?id=${currentMovieData.id}`;
        } else {
            console.error("Cannot open movie details: movie ID is missing.");
             alert("Could not open movie details."); // User feedback
        }
    }

    // --- Movie Details Page Trailer Modal ---
    const trailerModal = document.getElementById('trailer-modal');
    const trailerIframe = document.getElementById('trailer-iframe');
    const closeTrailerButton = document.querySelector('.close-trailer');

    if (trailerModal && trailerIframe) {
        window.playTrailer = function(trailerUrl) {
            if (trailerUrl) {
                // Determine if it's a YouTube URL or a file path
                if (trailerUrl.includes('youtube.com/watch') || trailerUrl.includes('youtu.be/')) {
                     // Convert YouTube watch URL to embed URL
                     let embedUrl = trailerUrl;
                     if (trailerUrl.includes('watch?v=')) {
                          const videoId = trailerUrl.split('v=')[1].split('&')[0];
                          embedUrl = `https://www.youtube.com/embed/${videoId}`;
                     } else if (trailerUrl.includes('youtu.be/')) {
                          const videoId = trailerUrl.split('/').pop();
                          embedUrl = `https://www.youtube.com/embed/${videoId}`;
                     }
                     // Add parameters for autoplay and related videos off
                     iframe.src = `${embedUrl}?autoplay=1&rel=0`;

                } else {
                    // Assuming it's a local file path or other embed type
                     // Need a video element for local files, not an iframe
                     // Or handle other embed formats (Vimeo etc.)
                     console.error("Unsupported trailer format:", trailerUrl);
                     alert("Unsupported trailer format. Cannot play.");
                     return; // Stop here if format unsupported
                }

                modal.classList.add('active');
            } else {
                 alert("Trailer not available for this movie.");
            }
        }

        window.closeTrailer = function() {
            // Stop the video by clearing the iframe src
            trailerIframe.src = '';
            trailerModal.classList.remove('active');
        }

        // Close modal when clicking outside
        trailerModal.addEventListener('click', function(e) {
            if (e.target === this) {
                closeTrailer();
            }
        });
        if (closeTrailerButton) {
             closeTrailerButton.addEventListener('click', closeTrailer);
        }
    }

    // --- Profile Page Edit Functionality ---
    window.toggleEdit = function(elementId) {
        const element = document.getElementById(elementId);
        if (!element) return;

        // Check if already in edit mode (has an input/textarea child)
        if (element.querySelector('.edit-input, .bio-input')) {
            return; // Already editing, do nothing
        }

        const currentText = element.textContent.trim();

        let input;
        if (elementId === 'bio') {
            input = document.createElement('textarea');
            input.className = 'edit-input bio-input';
            input.value = currentText === 'Click to add bio...' ? '' : currentText;
            input.placeholder = 'Write something about yourself...';
        } else {
            input = document.createElement('input');
            input.type = 'text';
            input.className = 'edit-input';
            input.value = currentText;
        }

        // Store the original text in case of cancellation or error (optional)
        // input.setAttribute('data-original-text', currentText);

        const saveChanges = async function() {
            let newValue = input.value.trim();
            const originalText = element.getAttribute('data-original-text') || currentText; // Use stored original or current

            // Restore element's original state temporarily
            element.textContent = originalText; // Show original text
            if(input.parentNode) {
                 input.parentNode.removeChild(input); // Remove the input field
            }


            if (elementId === 'bio' && !newValue) {
                newValue = 'Click to add bio...'; // Restore placeholder text if empty
            }

            // Only save if the value actually changed (and it's not just the bio placeholder)
            if (newValue !== originalText && (elementId !== 'bio' || newValue !== 'Click to add bio...')) {
                 console.log(`Saving ${elementId}: ${newValue}`);
                 // TODO: Make AJAX call to update_profile.php
                 try {
                      const formData = new FormData();
                      formData.append('field', elementId); // 'displayName', 'username', or 'bio'
                      formData.append('value', newValue);

                      const response = await fetch('update_profile.php', { // Adjust path
                           method: 'POST',
                           body: formData
                      });

                      const result = await response.json(); // Assuming JSON response {success: true/false, message: '...'}

                      if (result.success) {
                           console.log("Update successful:", result.message);
                           // Update the element with the new value from the server (server might clean/validate)
                           element.textContent = result.value || newValue; // Use server-returned value if available
                           // If username changed, potentially update username display elsewhere on the page if needed
                           if (elementId === 'displayName') {
                               // Update any other display name elements
                           } else if (elementId === 'username') {
                               // Update any other username elements
                               // Handle the '@' prefix visually if not stored in DB
                               const parentP = element.closest('.username');
                               if (parentP && !element.textContent.startsWith('@')) {
                                   // This might require restructuring the HTML slightly or handling the '@' purely visually via CSS pseudo-element or JS
                                   // For now, JS will just set the text. Make sure server doesn't save '@'.
                                   // Let's assume the server saves 'username' without '@' and JS adds/removes it visually if needed.
                                   // The original HTML has `@<span id="username">`. So JS should only update the span content.
                               }
                           }

                           // Update original text for subsequent edits
                           element.setAttribute('data-original-text', result.value || newValue);

                      } else {
                           console.error("Update failed:", result.message);
                           alert("Failed to save changes: " + result.message); // Provide user feedback
                           // Revert to original text on failure
                           element.textContent = originalText;
                           element.setAttribute('data-original-text', originalText); // Ensure original is restored
                      }
                 } catch (error) {
                      console.error("AJAX error saving profile:", error);
                      alert("An error occurred while saving changes."); // Generic error
                       element.textContent = originalText; // Revert on AJAX error
                       element.setAttribute('data-original-text', originalText);
                 }

            } else {
                 // Value didn't change, just restore the display
                 element.textContent = originalText;
                 element.setAttribute('data-original-text', originalText);
            }

             // Clean up event listeners if needed, though onblur handles it implicitly
        };

        // Add event listeners for blur (save) and keydown (Enter=save, Escape=cancel)
        input.addEventListener('blur', saveChanges);
        input.addEventListener('keydown', function(event) {
            if (event.key === 'Enter') {
                event.preventDefault(); // Prevent new line in textarea, form submission
                input.blur(); // Trigger blur to save
            } else if (event.key === 'Escape') {
                 // Restore original text and remove input
                 const originalText = element.getAttribute('data-original-text') || currentText;
                 element.textContent = originalText;
                  if(input.parentNode) {
                       input.parentNode.removeChild(input);
                  }
                 // Prevent default escape behavior
                 event.preventDefault();
            }
        });

        // Replace the element's content with the input field
        element.setAttribute('data-original-text', currentText); // Store original text
        element.textContent = ''; // Clear element content
        element.appendChild(input); // Add the input field
        input.focus(); // Focus the input field

         // For bio, remove the click listener temporarily while editing
         if (elementId === 'bio') {
              // This is tricky with inline onclick. Better to use addEventListener/removeEventListener
              // For now, the 'blur' event handles the transition back.
         }
    }

    // --- Favorite/Review Action Buttons ---
    // These are action buttons often found on movie cards (favorite icon, etc.)
    // We need to add event listeners dynamically or use delegation
    // Example for Favorite toggle (assuming a heart icon button with class 'action-btn favorite-btn')
    document.querySelectorAll('.action-btn.favorite-btn').forEach(button => {
        button.addEventListener('click', function(e) {
            e.stopPropagation(); // Prevent card click event from firing
            e.preventDefault(); // Prevent default button behavior

            const movieId = this.closest('.movie-card').getAttribute('data-movie-id'); // Get movie ID from a data attribute on the card
             if (!movieId) {
                  console.error("Movie ID not found on card for favorite toggle.");
                  return;
             }

             // Determine current state (is it currently favorited?) - maybe check icon class?
             const isFavorited = this.classList.contains('fas'); // Assuming fas means filled heart

             // TODO: Send AJAX request to favorite/toggle_favorite.php
             // Data: movieId, action (add/remove)
             fetch('favorite/toggle_favorite.php', { // Adjust path
                 method: 'POST',
                 headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                 body: `movie_id=${movieId}&action=${isFavorited ? 'remove' : 'add'}`
             })
             .then(response => response.json())
             .then(data => {
                 if (data.success) {
                     // Update icon visually
                     if (data.action === 'added') {
                         this.classList.remove('far');
                         this.classList.add('fas');
                         alert("Added to favorites!"); // User feedback
                     } else { // Removed
                         this.classList.remove('fas');
                         this.classList.add('far');
                         alert("Removed from favorites!"); // User feedback
                         // On the favorites page, you might want to remove the card
                          if (document.body.classList.contains('favorites-page')) {
                               this.closest('.movie-card').remove();
                               // Check if grid is now empty and show empty state
                               const grid = document.querySelector('.review-grid');
                               if (grid && grid.children.length === 0) {
                                    if (!document.querySelector('.empty-state:not(.search-empty-state)')) { // Check for default empty state
                                         const emptyState = document.createElement('div');
                                         emptyState.className = 'empty-state';
                                          emptyState.innerHTML = `
                                             <i class="fas fa-heart"></i>
                                             <p>No favorites yet</p>
                                             <p class="subtitle">Add movies to your favorites list to see them here</p>
                                         `; // Customize message
                                         grid.appendChild(emptyState);
                                    }
                               }
                          }
                     }
                 } else {
                     console.error("Failed to toggle favorite:", data.message);
                     alert("Failed to update favorites: " + data.message); // User feedback
                 }
             })
             .catch(error => {
                 console.error("AJAX Error toggling favorite:", error);
                 alert("An error occurred while updating favorites."); // Generic error
             });
        });
    });


}); // End DOMContentLoaded

```

**6. `index.php` (Baru - Root Redirect)**

Buat file `index.php` di root folder.

```php
<?php
// index.php - Redirect to beranda or login based on session

require_once __DIR__ . '/includes/auth_helper.php';

if (is_logged_in()) {
    header('Location: beranda/index.php');
    exit;
} else {
    header('Location: autentikasi/form-login.php');
    exit;
}
?>
```

**7. `autentikasi/form-login.php` (Modifikasi)**

*   Ubah path CSS (`../css/style.css`).
*   Ubah path JS (`../js/animation.js`).
*   Pastikan `require_once '../config/database.php';` ada.
*   Gunakan `getUserByUsernameOrEmail` untuk mencari user.
*   Sesuaikan variabel PHP untuk CAPTCHA agar sesuai dengan `config/database.php` dan `generateRandomString`.
*   Sertakan `includes/auth_helper.php` jika fungsi `is_logged_in` dll. diperlukan di sini (tapi sepertinya tidak, `config.php` sudah cukup).

```php
<?php
// autentikasi/form-login.php
// Include config.php - This should start session and include database functions
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php'; // Include auth helper

// Redirect to beranda if already logged in
if (is_logged_in()) {
    header('Location: ../beranda/index.php'); // Adjust path
    exit;
}

// --- Logika generate CAPTCHA (Server-side) ---
// Generate CAPTCHA baru jika belum ada di session atau jika ada error sebelumnya
// This should happen BEFORE any output
if (!isset($_SESSION['captcha_code'])) {
    $_SESSION['captcha_code'] = generateRandomString(6);
}

// --- Proses Form Login ---
$error_message = null;
$success_message = null;

// Check for messages passed via session from redirects (e.g., after registration)
if (isset($_SESSION['error'])) {
    $error_message = $_SESSION['error'];
    unset($_SESSION['error']);
}
if (isset($_SESSION['success'])) {
    $success_message = $_SESSION['success'];
    unset($_SESSION['success']);
}

// Handle POST request
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $username_input = trim($_POST['username'] ?? '');
    $password_input = $_POST['password'] ?? '';
    $captcha_input = trim($_POST['captcha_input'] ?? '');

    // --- Server-side Validation ---
    if (empty($username_input) || empty($password_input) || empty($captcha_input)) {
        $_SESSION['error'] = 'Username/Email, Password, dan CAPTCHA wajib diisi.';
        $_SESSION['captcha_code'] = generateRandomString(6); // Regenerate CAPTCHA on any error
        header('Location: form-login.php');
        exit;
    }

    // --- CAPTCHA Validation ---
    if (!isset($_SESSION['captcha_code']) || strtolower($captcha_input) !== strtolower($_SESSION['captcha_code'])) {
        $_SESSION['error'] = 'CAPTCHA Tidak Valid!';
        $_SESSION['captcha_code'] = generateRandomString(6); // Regenerate CAPTCHA on failure
        header('Location: form-login.php');
        exit;
    }

    // CAPTCHA valid, unset it
    unset($_SESSION['captcha_code']);

    // --- Authenticate User ---
    try {
        $user = getUserByUsernameOrEmail($username_input);

        if ($user && password_verify($password_input, $user['password'])) {
            // Authentication successful
            $_SESSION['user_id'] = $user['user_id'];
            session_regenerate_id(true);

            // Redirect user to the page they were trying to access or default beranda
            $redirect_url = $_SESSION['redirect_url'] ?? '../beranda/index.php'; // Adjust path
            unset($_SESSION['redirect_url']); // Clear stored URL
            header('Location: ' . $redirect_url);
            exit;

        } else {
            // Invalid credentials
            $_SESSION['error'] = 'Username/Email atau password salah.';
            $_SESSION['captcha_code'] = generateRandomString(6); // Regenerate CAPTCHA on failure
            header('Location: form-login.php');
            exit;
        }

    } catch (PDOException $e) {
        error_log("Database error during login: " . $e->getMessage());
        $_SESSION['error'] = 'Terjadi kesalahan internal saat login. Silakan coba lagi.';
        $_SESSION['captcha_code'] = generateRandomString(6); // Regenerate CAPTCHA on failure
        header('Location: form-login.php');
        exit;
    }
}

// Ambil kode CAPTCHA dari session untuk ditampilkan di client (setelah POST atau pada load awal)
$captchaCodeForClient = $_SESSION['captcha_code'] ?? ''; // Fallback

?>
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rate Tales - Login</title>
    <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png" sizes="16x16"> <!-- Adjust path -->
    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;600&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <!-- Google Sign-In - Placeholder -->
    <script src="https://accounts.google.com/gsi/client" async defer></script>
</head>
<body class="auth-page"> <!-- Add body class for specific page styling if needed -->
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
                <label for="username">Username atau Email</label>
                <input type="text" name="username" id="username" placeholder="Username atau Email" required value="<?php echo htmlspecialchars($_POST['username'] ?? ''); ?>">
            </div>
            <div class="input-group">
                <label for="password">Password</label>
                <input type="password" name="password" id="password" placeholder="Password" required>
            </div>

            <div class="remember-me">
                <input type="checkbox" name="remember_me" id="rememberMe">
                <label for="rememberMe">Ingat Saya</label>
            </div>

            <div class="input-group">
                 <label>Verifikasi CAPTCHA</label>
                 <div class="captcha-container">
                    <canvas id="captchaCanvas" width="150" height="40"></canvas>
                    <button type="button" onclick="generateCaptcha()" class="btn-reload" title="Reload CAPTCHA"><i class="fas fa-sync-alt"></i></button>
                 </div>
                 <input type="text" name="captcha_input" id="captchaInput" placeholder="Masukkan CAPTCHA" required autocomplete="off" value="<?php echo htmlspecialchars($_POST['captcha_input'] ?? ''); ?>">
                 <!-- Removed client-side message element as primary validation is server-side -->
            </div>

            <button type="submit" class="btn">Login</button>
        </form>
        <br>
        <!-- Google Sign-In elements -->
        <div id="g_id_onload" data-client_id="YOUR_GOOGLE_CLIENT_ID" data-callback="handleCredentialResponse"></div> <!-- Replace YOUR_GOOGLE_CLIENT_ID -->
        <div class="g_id_signin" data-type="standard"></div>

        <p class="form-link">Belum punya akun? <a href="form-register.php">Klik disini untuk register</a></p>
    </div>

    <script src="../js/animation.js"></script> <!-- Adjusted JS path -->
    <script>
        // Variable to store the current CAPTCHA code from the server
        let currentCaptchaCode = "<?php echo $captchaCodeForClient; ?>";

        const captchaInput = document.getElementById('captchaInput');
        const captchaCanvas = document.getElementById('captchaCanvas');


        function drawCaptcha(code) {
            if (!captchaCanvas) return;
            const ctx = captchaCanvas.getContext('2d');
            ctx.clearRect(0, 0, captchaCanvas.width, captchaCanvas.height);
            // Use colors from the main CSS file or define here
            ctx.fillStyle = "#1a1a1a"; // Background matching input
            ctx.fillRect(0, 0, captchaCanvas.width, captchaCanvas.height);
            ctx.font = "24px Arial";
            ctx.fillStyle = "#00ffff"; // Text color
            ctx.strokeStyle = "#363636"; // Random line color matching border
            ctx.lineWidth = 1;

            // Add random lines
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

        // Function to generate new CAPTCHA using Fetch
        async function generateCaptcha() {
            try {
                // Adjust path to generate_captcha.php relative to form-login.php
                const response = await fetch('generate_captcha.php');

                if (!response.ok) {
                     throw new Error('Failed to load new CAPTCHA (status: ' + response.status + ')');
                 }

                const newCaptchaCode = await response.text();
                currentCaptchaCode = newCaptchaCode; // Update client-side variable
                drawCaptcha(currentCaptchaCode); // Draw the new CAPTCHA

                captchaInput.value = ''; // Clear input field
                // No client-side message needed on reload
                // captchaMessage.style.display = 'none';
                // captchaMessage.innerText = '';

            } catch (error) {
                console.error("Error generating CAPTCHA:", error);
                // Display an error message if CAPTCHA fails to load
                // captchaMessage.innerText = 'Gagal memuat CAPTCHA. Coba lagi.';
                // captchaMessage.style.color = 'red';
                // captchaMessage.style.display = 'block';
                alert("Gagal memuat CAPTCHA. Silakan refresh halaman."); // Simple alert feedback
            }
        }

        // Initial drawing of CAPTCHA when the page loads
        document.addEventListener('DOMContentLoaded', () => {
            drawCaptcha(currentCaptchaCode);
        });

        // Google Sign-In (Placeholder)
        function handleCredentialResponse(response) {
           console.log("Encoded JWT ID token: " + response.credential);
           // TODO: Send token to server for validation
           // window.location.href = 'verify-google-token.php?token=' + response.credential;
        }
    </script>
</body>
</html>

```

**8. `autentikasi/form-register.php` (Modifikasi)**

*   Ubah path CSS (`../css/style.css`).
*   Ubah path JS (`../js/animation.js`).
*   Pastikan `require_once '../config/database.php';` ada.
*   Gunakan `createUser` untuk menyimpan user baru.
*   Sesuaikan variabel PHP untuk CAPTCHA.
*   Perbaiki tag `<button>` untuk "Baca Perjanjian Pengguna".

```php
<?php
// autentikasi/form-register.php
// Include config.php - This should start session and include database functions
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php'; // Include auth helper

// Redirect to beranda if already logged in
if (is_logged_in()) {
    header('Location: ../beranda/index.php'); // Adjust path
    exit;
}

// --- Logika generate CAPTCHA (Server-side) ---
if (!isset($_SESSION['captcha_code'])) {
    $_SESSION['captcha_code'] = generateRandomString(6);
}

// --- Proses Form Register ---
$error_message = null;
$success_message = null;

// Check for messages passed via session from redirects
if (isset($_SESSION['error'])) {
    $error_message = $_SESSION['error'];
    unset($_SESSION['error']);
}
if (isset($_SESSION['success'])) {
    $success_message = $_SESSION['success'];
    unset($_SESSION['success']);
}


// Handle POST request
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // Ambil dan bersihkan input
    $full_name = trim($_POST['full_name'] ?? '');
    $username = trim($_POST['username'] ?? '');
    $age_input = $_POST['age'] ?? '';
    $gender = $_POST['gender'] ?? '';
    $email = trim($_POST['email'] ?? '');
    $password = $_POST['password'] ?? '';
    $confirm_password = $_POST['confirm_password'] ?? '';
    $captcha_input = $_POST['captcha_input'] ?? '';
    $agree = isset($_POST['agree']);

    // --- Server-side Validation ---
    $errors = [];

    if (empty($full_name)) $errors[] = 'Nama Lengkap wajib diisi.';
    if (empty($username)) $errors[] = 'Username wajib diisi.';
    if (empty($email)) $errors[] = 'Email wajib diisi.';
    if (empty($password)) $errors[] = 'Password wajib diisi.';
    if (empty($confirm_password)) $errors[] = 'Konfirmasi Password wajib diisi.';
    if (empty($captcha_input)) $errors[] = 'CAPTCHA wajib diisi.';
    if (!$agree) $errors[] = 'Anda harus menyetujui Perjanjian Pengguna.';

    // Validate Age
    $age = filter_var($age_input, FILTER_VALIDATE_INT);
    if ($age === false || $age <= 0 || $age > 120) { // Added max age limit
        $errors[] = 'Usia tidak valid.';
    }

    // Validate Gender
    $allowed_genders = ['Laki-laki', 'Perempuan'];
    if (!in_array($gender, $allowed_genders)) {
        $errors[] = 'Jenis Kelamin tidak valid.';
    }

    // Validate Password Match
    if ($password !== $confirm_password) {
        $errors[] = 'Password dan konfirmasi tidak cocok.';
    }

    // Validate Password Length
    if (strlen($password) < 6) {
        $errors[] = 'Password minimal harus 6 karakter.';
    }

    // --- CAPTCHA Validation (Primary Server-side) ---
    if (!isset($_SESSION['captcha_code']) || strtolower($captcha_input) !== strtolower($_SESSION['captcha_code'])) {
        $errors[] = 'CAPTCHA Tidak Valid!';
         // IMPORTANT: Regenerate CAPTCHA on failure regardless of other errors
         $_SESSION['captcha_code'] = generateRandomString(6);
    } else {
        // CAPTCHA is valid for this submission, unset it so it can't be reused
        unset($_SESSION['captcha_code']);
    }


    // If validation fails, set error message and redirect
    if (!empty($errors)) {
        $_SESSION['error'] = implode('<br>', $errors);
         // Ensure a new CAPTCHA is always generated for the next attempt if it wasn't already regenerated above
         if (!isset($_SESSION['captcha_code'])) {
              $_SESSION['captcha_code'] = generateRandomString(6);
         }
        header('Location: form-register.php');
        exit;
    }

    // --- If all validation passes, check uniqueness and create user ---

    try {
        // Check if username or email already exists
        $stmt_check = $pdo->prepare("SELECT COUNT(*) FROM users WHERE username = ? OR email = ?");
        $stmt_check->execute([$username, $email]);
        if ($stmt_check->fetchColumn() > 0) {
            $_SESSION['error'] = 'Username atau email sudah terdaftar.';
            $_SESSION['captcha_code'] = generateRandomString(6); // Regenerate CAPTCHA on DB check failure
            header('Location: form-register.php');
            exit;
        }

        // Create user using the function
        $success = createUser($full_name, $username, $email, $password, $age, $gender);

        if ($success) {
            // Get the user ID of the newly created user for direct login
             $stmt_get_user = $pdo->prepare("SELECT user_id FROM users WHERE username = ?");
             $stmt_get_user->execute([$username]);
             $newUser = $stmt_get_user->fetch();

             if ($newUser) {
                 $_SESSION['user_id'] = $newUser['user_id'];
                 session_regenerate_id(true); // Session Fixation Protection

                 $_SESSION['success'] = 'Pendaftaran berhasil! Selamat datang, ' . htmlspecialchars($username) . '!';
                 header('Location: ../beranda/index.php'); // Redirect to beranda
                 exit;
             } else {
                 // Should not happen if createUser was successful, but handle defensively
                 $_SESSION['success'] = 'Pendaftaran berhasil. Silakan login.'; // Less personalized message
                 header('Location: form-login.php'); // Redirect to login instead
                 exit;
             }


        } else {
            // This might happen if there's a generic DB error not caught by exception
            $_SESSION['error'] = 'Pendaftaran gagal. Silakan coba lagi.';
            $_SESSION['captcha_code'] = generateRandomString(6); // Regenerate CAPTCHA on failure
            header('Location: form-register.php');
            exit;
        }

    } catch (PDOException $e) {
        error_log("Database error during registration: " . $e->getMessage());
        $_SESSION['error'] = 'Terjadi kesalahan internal saat pendaftaran. Silakan coba lagi.';
        $_SESSION['captcha_code'] = generateRandomString(6); // Regenerate CAPTCHA on failure
        header('Location: form-register.php');
        exit;
    }
}

// Ambil kode CAPTCHA dari session untuk ditampilkan di client (setelah POST atau pada load awal)
$captchaCodeForClient = $_SESSION['captcha_code'] ?? '';

?>
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rate Tales - Daftar</title>
    <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;600&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
</head>
<body class="auth-page"> <!-- Add body class -->
    <div class="form-container register-form">
        <h2>Daftar Akun</h2>

        <?php if ($error_message): ?>
            <p class="error-message"><?php echo htmlspecialchars($error_message); ?></p>
        <?php endif; ?>

         <?php if ($success_message): ?>
            <p class="success-message"><?php echo htmlspecialchars($success_message); ?></p>
        <?php endif; ?>

        <form method="POST" action="form-register.php" onsubmit="return validateForm()">
            <div class="input-group">
                <label for="name">Nama Lengkap</label>
                <input type="text" id="name" name="full_name" placeholder="Masukkan nama lengkap" required value="<?php echo htmlspecialchars($_POST['full_name'] ?? ''); ?>">
            </div>
            <div class="input-group">
                <label for="username">Username</label>
                <input type="text" id="username" name="username" placeholder="Masukkan username" required value="<?php echo htmlspecialchars($_POST['username'] ?? ''); ?>">
            </div>
            <div class="input-group">
                <label for="usia">Usia</label>
                <input type="number" id="usia" name="age" placeholder="Usia anda saat ini" required min="1" value="<?php echo htmlspecialchars($_POST['age'] ?? ''); ?>">
            </div>
            <div class="input-group">
                <label for="gender">Jenis Kelamin</label>
                <select id="gender" name="gender" required>
                    <option value="">Pilih...</option>
                    <option value="Laki-laki" <?php echo (($_POST['gender'] ?? '') === 'Laki-laki') ? 'selected' : ''; ?>>Laki-laki</option>
                    <option value="Perempuan" <?php echo (($_POST['gender'] ?? '') === 'Perempuan') ? 'selected' : ''; ?>>Perempuan</option>
                </select>
            </div>
            <div class="input-group">
                <label for="email">Email</label>
                <input type="email" id="email" name="email" placeholder="Masukkan email" required value="<?php echo htmlspecialchars($_POST['email'] ?? ''); ?>">
            </div>
            <div class="input-group">
                <label for="password">Buat Password</label>
                <input type="password" id="password" name="password" placeholder="Buat password anda" required minlength="6">
            </div>
            <div class="input-group">
                <label for="confirm-password">Konfirmasi Password</label>
                <input type="password" id="confirm-password" name="confirm_password" placeholder="Ulangi password" required>
            </div>

            <!-- CAPTCHA HTML -->
            <div class="input-group">
                 <label>Verifikasi CAPTCHA</label>
                 <div class="captcha-container">
                    <canvas id="captchaCanvas" width="150" height="40"></canvas>
                    <button type="button" onclick="generateCaptcha()" class="btn-reload" title="Reload CAPTCHA"><i class="fas fa-sync-alt"></i></button>
                 </div>
                 <input type="text" name="captcha_input" id="captchaInput" placeholder="Masukkan CAPTCHA" required autocomplete="off" value="<?php echo htmlspecialchars($_POST['captcha_input'] ?? ''); ?>">
                 <p id="captchaMessage" class="error-message" style="display:none;"></p> <!-- For client-side password match feedback -->
            </div>

            <div style="text-align: center; margin-bottom: 10px; margin-top: 20px;">
                <button type="button" id="agreement-btn">Baca Perjanjian Pengguna</button> <!-- Use button tag -->
            </div>

            <div class="input-group agreement-checkbox">
                <input type="checkbox" id="agree-checkbox" name="agree" required <?php echo (isset($_POST['agree']) && $_POST['agree']) ? 'checked' : ''; ?>>
                <label for="agree-checkbox">Saya menyetujui <a href="#" id="show-agreement-link">Perjanjian Pengguna</a></label>
            </div>

            <button type="submit" class="btn" id="register-submit-btn">Daftar</button>
        </form>
        <p class="form-link">Sudah punya akun? <a href="form-login.php">Login di sini</a></p>
    </div>

    <!-- Agreement Modal HTML -->
    <div id="agreement-modal">
        <div>
            <h3>Perjanjian Pengguna</h3>
            <p>
                <h5><b>Kebijakan Privasi</b></h5>
                                Kebijakan Privasi ini menjelaskan bagaimana Rate Tales (“kami”) mengumpulkan, menyimpan, menggunakan, dan melindungi data pribadi Anda selama Anda menggunakan situs ini. Seluruh aktivitas pengelolaan data dilakukan sesuai dengan ketentuan Undang-Undang Republik Indonesia Nomor 27 Tahun 2022 tentang Perlindungan Data Pribadi (UU PDP). Dengan menggunakan situs ini dan mendaftarkan akun Anda, Anda memberikan persetujuan eksplisit kepada kami untuk memproses data pribadi Anda sebagaimana dijelaskan dalam kebijakan ini.
                Kami dapat mengumpulkan informasi pribadi secara langsung saat Anda mendaftar atau menggunakan fitur di situs, seperti nama lengkap, alamat email, serta informasi terkait aktivitas Anda di situs ini, termasuk preferensi tontonan, ulasan, rating, dan riwayat interaksi. Semua data yang kami kumpulkan digunakan untuk tujuan yang sah dan proporsional, yakni untuk meningkatkan pengalaman Anda dalam menggunakan layanan kami. Kami menggunakannya untuk menyediakan fitur-fitur yang dipersonalisasi, memberikan rekomendasi konten, melakukan analisis internal, serta—dengan persetujuan Anda—menyampaikan informasi promosi atau konten yang relevan.
                Data pribadi Anda akan disimpan selama akun Anda masih aktif, atau selama diperlukan untuk mendukung tujuan layanan. Kami menerapkan langkah-langkah teknis dan organisasi yang sesuai untuk melindungi data Anda dari akses yang tidak sah, kebocoran, atau penyalahgunaan. Kami tidak akan membagikan data pribadi Anda kepada pihak ketiga tanpa persetujuan eksplisit dari Anda, kecuali jika diharuskan oleh hukum atau dalam konteks penegakan hukum dan kewajiban hukum lainnya.
                Sesuai dengan ketentuan UU PDP, Anda sebagai pemilik data memiliki hak untuk mengakses data pribadi Anda, meminta perbaikan atau penghapusan data, menarik kembali persetujuan atas pemprosesan data, serta mengajukan keberatan atas pemprosesan tertentu. Kami menghormati hak-hak tersebut dan akan menindaklanjuti setiap permintaan yang Anda sampaikan melalui saluran kontak resmi yang tersedia di situs kami.
                Kami dapat memperbarui isi Kebijakan Privasi ini dari waktu ke waktu, terutama jika terjadi perubahan peraturan atau perkembangan teknologi yang memengaruhi cara kami memproses data pribadi Anda. Perubahan signifikan akan kami sampaikan melalui notifikasi di situs atau email. Dengan terus menggunakan layanan kami setelah perubahan diberlakukan, Anda dianggap telah menyetujui kebijakan yang diperbarui.
                Jika Anda memiliki pertanyaan, permintaan, atau keluhan terkait kebijakan ini atau penggunaan data pribadi Anda, Anda dapat menghubungi kami melalui alamat email atau formulir kontak resmi yang tersedia di situs. Dengan menggunakan situs ini, Anda menyatakan telah membaca, memahami, dan menyetujui isi Kebijakan Privasi ini serta memberikan persetujuan eksplisit atas pengumpulan dan pemprosesan data pribadi Anda oleh kami.
            </p>
            <button id="close-agreement" class="btn">Tutup</button>
        </div>
    </div>

    <script src="../js/animation.js"></script> <!-- Adjusted JS path -->
    <script>
        // Variable to store the current CAPTCHA code from the server
        let currentCaptchaCode = "<?php echo $captchaCodeForClient; ?>";

        const captchaInput = document.getElementById('captchaInput');
        const captchaMessage = document.getElementById('captchaMessage');
        const captchaCanvas = document.getElementById('captchaCanvas');


        function drawCaptcha(code) {
            if (!captchaCanvas) return;
            const ctx = captchaCanvas.getContext('2d');
            ctx.clearRect(0, 0, captchaCanvas.width, captchaCanvas.height);
            // Use colors from the main CSS file or define here
            ctx.fillStyle = "#1a1a1a"; // Background matching input
            ctx.fillRect(0, 0, captchaCanvas.width, captchaCanvas.height);
            ctx.font = "24px Arial";
            ctx.fillStyle = "#00ffff"; // Text color
            ctx.strokeStyle = "#363636"; // Random line color matching border
            ctx.lineWidth = 1;

            // Add random lines
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

        // Function to generate new CAPTCHA using Fetch
        async function generateCaptcha() {
            try {
                 // Adjust path to generate_captcha.php relative to form-register.php
                const response = await fetch('generate_captcha.php');

                if (!response.ok) {
                     throw new Error('Failed to load new CAPTCHA (status: ' + response.status + ')');
                 }

                const newCaptchaCode = await response.text();
                currentCaptchaCode = newCaptchaCode;
                drawCaptcha(currentCaptchaCode);

                captchaInput.value = ''; // Clear input field
                captchaMessage.style.display = 'none'; // Clear messages
                captchaMessage.innerText = '';

            } catch (error) {
                console.error("Error generating CAPTCHA:", error);
                // Display an error message if CAPTCHA fails to load
                 captchaMessage.innerText = 'Gagal memuat CAPTCHA. Coba lagi.';
                 captchaMessage.style.color = 'red';
                 captchaMessage.style.display = 'block';
            }
        }

        // Initial drawing of CAPTCHA when the page loads
        document.addEventListener('DOMContentLoaded', () => {
            drawCaptcha(currentCaptchaCode);

            // Set initial state of Register button based on agreement checkbox
            const agreeCheckbox = document.getElementById('agree-checkbox');
            const registerSubmitBtn = document.getElementById('register-submit-btn');
            if(registerSubmitBtn && agreeCheckbox) {
                registerSubmitBtn.disabled = !agreeCheckbox.checked;
            }
        });


        // Client-side form validation (for password match and agreement)
        function validateForm() {
            const password = document.getElementById('password').value;
            const confirmPassword = document.getElementById('confirm-password').value;
            const agreeCheckbox = document.getElementById('agree-checkbox');

            const validationMessageElement = captchaMessage; // Use captchaMessage element for general form validation feedback

            validationMessageElement.style.display = 'none'; // Hide previous messages
            validationMessageElement.innerText = '';
            validationMessageElement.style.color = 'red'; // Reset color for errors

            if (password !== confirmPassword) {
                 validationMessageElement.innerText = 'Password dan Konfirmasi Password tidak cocok!';
                 validationMessageElement.style.display = 'block';
                return false;
            }

             if (!agreeCheckbox.checked) {
                 validationMessageElement.innerText = 'Anda harus menyetujui Perjanjian Pengguna.';
                 validationMessageElement.style.display = 'block';
                 return false;
            }

            // CAPTCHA validation is handled server-side.
            // If client-side checks pass, the form submits, and server validates CAPTCHA + other data.
            return true;
        }

        // Modal Agreement
        const agreementBtn = document.getElementById('agreement-btn');
        const showAgreementLink = document.getElementById('show-agreement-link');
        const agreementModal = document.getElementById('agreement-modal');
        const closeAgreement = document.getElementById('close-agreement');
        const agreeCheckbox = document.getElementById('agree-checkbox');
        const registerSubmitBtn = document.getElementById('register-submit-btn');

        function showAgreementModal() {
            if(agreementModal) agreementModal.style.display = 'flex';
        }

        if(agreementBtn) agreementBtn.addEventListener('click', showAgreementModal);
        if(showAgreementLink) showAgreementLink.addEventListener('click', (e) => {
            e.preventDefault();
            showAgreementModal();
        });
        if(closeAgreement) closeAgreement.addEventListener('click', () => {
            if(agreementModal) agreementModal.style.display = 'none';
        });

        if(agreementModal) {
            agreementModal.addEventListener('click', (e) => {
                if (e.target === agreementModal) {
                    agreementModal.style.display = 'none';
                }
            });
        }

        // Enable/disable Register button
        if(agreeCheckbox && registerSubmitBtn) {
            agreeCheckbox.addEventListener('change', () => {
                registerSubmitBtn.disabled = !agreeCheckbox.checked;
            });
        }

    </script>
</body>
</html>
```

**9. `autentikasi/generate_captcha.php` (Pindah & Modifikasi)**

Pindahkan file ini ke folder `autentikasi/`. Pastikan path ke `config.php` benar.

```php
<?php
// autentikasi/generate_captcha.php
// File ini hanya bertugas menghasilkan kode CAPTCHA baru dan menyimpannya di session

// Include file konfigurasi untuk koneksi DB, session, dan helper function
// config.php seharusnya sudah memulai session dan mendefinisikan generateRandomString()
require_once __DIR__ . '/../config/database.php';

// Pastikan sesi sudah aktif (seharusnya sudah di config.php)
if (session_status() == PHP_SESSION_NONE) {
    session_start();
}

// Hasilkan kode CAPTCHA baru menggunakan fungsi dari config.php
$newCaptchaCode = generateRandomString(6); // Length 6 as used in forms

// Simpan kode baru di session, menimpa yang lama
$_SESSION['captcha_code'] = $newCaptchaCode;

// Set header untuk memberitahu klien bahwa ini adalah plain text
header('Content-Type: text/plain');
header('Cache-Control: no-cache, no-store, must-revalidate'); // Prevent caching
header('Pragma: no-cache');
header('Expires: 0');

// Output kode CAPTCHA baru ke klien
echo $newCaptchaCode;

// Hentikan eksekusi skrip
exit;
?>
```

**10. `autentikasi/logout.php` (Pindah & Modifikasi)**

Pindahkan file ini ke folder `autentikasi/`. Pastikan path ke `config.php` benar.

```php
<?php
// autentikasi/logout.php
require_once __DIR__ . '/../config/database.php';

if (session_status() == PHP_SESSION_NONE) {
    session_start();
}

// Unset all of the session variables.
$_SESSION = array();

// If it's desired to kill the session, also delete the session cookie.
// Note: This will destroy the session, and not just the session data!
if (ini_get("session.use_cookies")) {
    $params = session_get_cookie_params();
    setcookie(session_name(), '', time() - 42000,
        $params["path"], $params["domain"],
        $params["secure"], $params["httponly"]
    );
}

// Finally, destroy the session.
session_destroy();

// Redirect to the login page
header('Location: form-login.php'); // Adjust path as needed
exit;
?>
```

**11. `beranda/index.php` (Modifikasi)**

*   Ubah path CSS (`../css/style.css`).
*   Ubah path JS (`../js/main.js`).
*   Sertakan `config/database.php` dan `includes/auth_helper.php`.
*   Ambil data film dari database menggunakan `getAllMovies()`.
*   Gunakan data film dari DB untuk slider dan grid.
*   Ambil data user yang login untuk menampilkan di pojok kanan atas.
*   Perbaiki path link sidebar.

```php
<?php
// beranda/index.php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php';

// Redirect to login if not logged in
require_login();

// Get logged-in user data
$loggedInUser = get_logged_in_user_data();

// Fetch movies from the database
$movies = getAllMovies();

// Take the first few movies for the slider (assuming getAllMovies orders by creation date desc)
$featuredMovies = array_slice($movies, 0, min(5, count($movies))); // Get up to 5 for slider

// For trending/for you, just reuse the $movies list for now as original did
$trendingMovies = $movies;
$forYouMovies = array_slice($movies, 0, min(8, count($movies))); // Limit for example

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Home</title>
    <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">
</head>
<body class="home-page"> <!-- Add body class -->
    <div class="container">
        <nav class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li class="active"><a href="#"><i class="fas fa-home"></i> <span>Home</span></a></li>
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favorites</span></a></li> <!-- Adjusted path -->
                <li><a href="../review/index.php"><i class="fas fa-star"></i> <span>Review</span></a></li> <!-- Adjusted path -->
                <li><a href="../manage/index.php"><i class="fas fa-film"></i> <span>Manage</span></a></li> <!-- Adjusted path -->
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Adjusted path -->
                </ul>
            </div>
        </nav>
        <main class="main-content">
            <div class="hero-section">
                <div class="featured-movie-slider">
                    <?php if (!empty($featuredMovies)): ?>
                        <?php foreach ($featuredMovies as $index => $movie):
                             // Get year from release_date
                             $releaseYear = date('Y', strtotime($movie['release_date']));
                             // Join genres array into a string
                             $genresString = !empty($movie['genres']) ? implode(', ', $movie['genres']) : 'N/A';
                         ?>
                        <div class="slide <?php echo $index === 0 ? 'active' : ''; ?>">
                            <img src="<?php echo htmlspecialchars($movie['poster_image']); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                            <div class="movie-info">
                                <h1><?php echo htmlspecialchars($movie['title']); ?></h1>
                                <p><?php echo htmlspecialchars($releaseYear); ?> | <?php echo htmlspecialchars($genresString); ?></p>
                            </div>
                        </div>
                        <?php endforeach; ?>
                    <?php else: ?>
                        <div class="empty-state slide active" style="position:relative; height: 100%; display: flex; flex-direction: column; justify-content: center; align-items: center; text-align: center; background-color: #242424;">
                            <i class="fas fa-film" style="font-size: 4em; color: #00ffff; margin-bottom: 20px;"></i>
                            <p style="font-size: 1.2em;">No featured movies available yet</p>
                            <p class="subtitle" style="color: #666; font-size: 0.9em;">Content will appear here when movies are added.</p>
                        </div>
                    <?php endif; ?>
                </div>
            </div>
            <section class="trending-section">
                <h2>TRENDING</h2>
                <div class="scroll-container">
                    <div class="movie-grid">
                         <?php if (!empty($trendingMovies)): ?>
                            <?php foreach ($trendingMovies as $movie):
                                $releaseYear = date('Y', strtotime($movie['release_date']));
                                $genresString = !empty($movie['genres']) ? implode(', ', $movie['genres']) : 'N/A';
                            ?>
                            <div class="movie-card" data-movie-id="<?php echo $movie['movie_id']; ?>" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'"> <!-- Link to details page -->
                                <img src="<?php echo htmlspecialchars($movie['poster_image']); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                <div class="movie-details">
                                    <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                                    <p><?php echo htmlspecialchars($releaseYear); ?> | <?php echo htmlspecialchars($genresString); ?></p>
                                </div>
                            </div>
                            <?php endforeach; ?>
                         <?php else: ?>
                             <div class="empty-state" style="min-width: 300px;">
                                 <i class="fas fa-film"></i>
                                 <p>No trending movies found</p>
                             </div>
                         <?php endif; ?>
                    </div>
                </div>
            </section>
            <section class="for-you-section">
                <h2>For You</h2>
                <div class="scroll-container">
                    <div class="movie-grid">
                         <?php if (!empty($forYouMovies)): ?>
                            <?php foreach ($forYouMovies as $movie):
                                 $releaseYear = date('Y', strtotime($movie['release_date']));
                                $genresString = !empty($movie['genres']) ? implode(', ', $movie['genres']) : 'N/A';
                             ?>
                            <div class="movie-card" data-movie-id="<?php echo $movie['movie_id']; ?>" onclick="window.location.href='../review/movie-details.php?id=<?php echo $movie['movie_id']; ?>'"> <!-- Link to details page -->
                                <img src="<?php echo htmlspecialchars($movie['poster_image']); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                                <div class="movie-details">
                                    <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                                    <p><?php echo htmlspecialchars($releaseYear); ?> | <?php echo htmlspecialchars($genresString); ?></p>
                                </div>
                            </div>
                            <?php endforeach; ?>
                         <?php else: ?>
                            <div class="empty-state" style="min-width: 300px;">
                                <i class="fas fa-film"></i>
                                <p>No movies for you found</p>
                             </div>
                         <?php endif; ?>
                    </div>
                </div>
            </section>
        </main>
        <div class="user-profile" onclick="window.location.href='../acc_page/index.php'"> <!-- Link to profile page -->
            <img src="<?php echo htmlspecialchars($loggedInUser['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($loggedInUser['username'] ?? 'User') . '&background=random'); ?>" alt="User Profile" class="profile-pic">
            <span><?php echo htmlspecialchars($loggedInUser['username'] ?? 'Guest'); ?></span>
        </div>
    </div>
    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
</body>
</html>

```

**12. `favorite/index.php` (Modifikasi)**

*   Ubah path CSS (`../css/style.css`).
*   Ubah path JS (`../js/main.js`).
*   Sertakan `config/database.php` dan `includes/auth_helper.php`.
*   Ambil film favorit pengguna yang login dari database menggunakan `getUserFavorites()`.
*   Gunakan data film dari DB untuk menampilkan card.
*   Implementasikan action button "Remove" (AJAX endpoint).
*   Perbaiki path link sidebar.

```php
<?php
// favorite/index.php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php';

// Redirect to login if not logged in
require_login();

$userId = get_user_id();
$favoriteMovies = getUserFavorites($userId);

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Favorites</title>
     <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">
</head>
<body class="favorites-page"> <!-- Add body class for JS targeting -->
    <div class="container">
        <nav class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li> <!-- Adjusted path -->
                <li class="active"><a href="#"><i class="fas fa-heart"></i> <span>Favorites</span></a></li>
                <li><a href="../review/index.php"><i class="fas fa-star"></i> <span>Review</span></a></li> <!-- Adjusted path -->
                <li><a href="../manage/index.php"><i class="fas fa-film"></i> <span>Manage</span></a></li> <!-- Adjusted path -->
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Adjusted path -->
                </ul>
            </div>
        </nav>
        <main class="main-content">
            <div class="review-header">
                <h1>My Favorites</h1>
                <div class="search-bar">
                    <input type="text" placeholder="Search favorites...">
                    <button><i class="fas fa-search"></i></button>
                </div>
            </div>
            <div class="review-grid">
                <?php if (!empty($favoriteMovies)): ?>
                    <?php foreach ($favoriteMovies as $movie):
                        $releaseYear = date('Y', strtotime($movie['release_date']));
                        $genresString = !empty($movie['genres']) ? implode(', ', $movie['genres']) : 'N/A';
                        // Average rating was fetched in getUserFavorites
                        $averageRating = $movie['average_rating'];
                    ?>
                    <div class="movie-card" data-movie-id="<?php echo $movie['movie_id']; ?>">
                        <div class="movie-poster">
                            <img src="<?php echo htmlspecialchars($movie['poster_image']); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                            <div class="movie-actions">
                                <button class="action-btn favorite-btn fas fa-trash" title="Remove from favorites"></button> <!-- Use trash icon -->
                            </div>
                        </div>
                        <div class="movie-details">
                            <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                            <p class="movie-info"><?php echo htmlspecialchars($releaseYear); ?> | <?php echo htmlspecialchars($genresString); ?></p>
                            <div class="rating">
                                <div class="stars">
                                    <?php
                                    $rating = floatval($averageRating); // Use fetched average rating
                                    for ($i = 1; $i <= 5; $i++) {
                                        if ($i <= $rating) {
                                            echo '<i class="fas fa-star"></i>';
                                        } else if ($i - 0.5 <= $rating) {
                                            echo '<i class="fas fa-star-half-alt"></i>';
                                        } else {
                                            echo '<i class="far fa-star"></i>';
                                        }
                                    }
                                    ?>
                                </div>
                                <span class="rating-count">(<?php echo htmlspecialchars($averageRating); ?>)</span>
                            </div>
                        </div>
                    </div>
                    <?php endforeach; ?>
                <?php else: ?>
                    <div class="empty-state">
                        <i class="fas fa-heart"></i>
                        <p>No favorites yet</p>
                        <p class="subtitle">Add movies to your favorites list to see them here</p>
                    </div>
                <?php endif; ?>
            </div>
        </main>
    </div>
    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
</body>
</html>
```

**13. `manage/index.php` (Modifikasi)**

*   Ubah path CSS (`../css/style.css`).
*   Ubah path JS (`../js/main.js`).
*   Sertakan `config/database.php` dan `includes/auth_helper.php`.
*   Ambil film yang diunggah pengguna yang login menggunakan `getMoviesByUserId()`.
*   Tampilkan film dalam grid atau pesan "no movies uploaded yet".
*   Perbaiki path link sidebar dan upload button.

```php
<?php
// manage/index.php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php';

// Redirect to login if not logged in
require_login();

$userId = get_user_id();
$uploadedMovies = getMoviesByUserId($userId);

?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Manage Movies - RatingTales</title>
     <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
</head>
<body class="manage-page"> <!-- Add body class for JS targeting -->
    <div class="container">
        <!-- Sidebar -->
        <div class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li> <!-- Adjusted path -->
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favorites</span></a></li> <!-- Adjusted path -->
                <li><a href="../review/index.php"><i class="fas fa-star"></i> <span>Review</span></a></li> <!-- Adjusted path -->
                <li class="active"><a href="#"><i class="fas fa-film"></i> <span>Manage</span></a></li>
            </ul>
            <ul class="bottom-links">
                <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Adjusted path -->
            </ul>
        </div>

        <!-- Main Content -->
        <main class="main-content">
            <div class="header">
                <div class="search-bar">
                    <i class="fas fa-search"></i>
                    <input type="text" placeholder="Search movies...">
                </div>
                 <!-- No user profile on manage page in original code, keep it simple -->
                 <!-- <div class="user-profile"> ... </div> -->
            </div>

            <!-- Movies Grid -->
            <div class="movies-grid">
                <?php if (empty($uploadedMovies)): ?>
                    <div class="empty-state">
                        <i class="fas fa-film"></i>
                        <p>No movies uploaded yet</p>
                        <p class="subtitle">Start adding your movies by clicking the upload button below</p>
                    </div>
                <?php else: ?>
                     <?php foreach ($uploadedMovies as $movie):
                          $releaseYear = date('Y', strtotime($movie['release_date']));
                          $genresString = !empty($movie['genres']) ? implode(', ', $movie['genres']) : 'N/A';
                          $averageRating = getAverageMovieRating($movie['movie_id']); // Fetch average rating
                     ?>
                     <div class="movie-card" data-movie-id="<?php echo $movie['movie_id']; ?>">
                          <div class="movie-poster">
                              <img src="<?php echo htmlspecialchars($movie['poster_image']); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                              <div class="movie-actions">
                                   <!-- Edit Button Placeholder -->
                                   <button class="action-btn edit-btn" title="Edit Movie"><i class="fas fa-edit"></i></button>
                                   <!-- Delete Button Placeholder -->
                                   <button class="action-btn delete-btn" title="Delete Movie"><i class="fas fa-trash"></i></button>
                              </div>
                         </div>
                         <div class="movie-details">
                             <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                             <p class="movie-info"><?php echo htmlspecialchars($releaseYear); ?> | <?php echo htmlspecialchars($genresString); ?></p>
                              <div class="rating">
                                  <div class="stars">
                                      <?php
                                      $rating = floatval($averageRating);
                                      for ($i = 1; $i <= 5; $i++) {
                                          if ($i <= $rating) {
                                              echo '<i class="fas fa-star"></i>';
                                          } else if ($i - 0.5 <= $rating) {
                                              echo '<i class="fas fa-star-half-alt"></i>';
                                          } else {
                                              echo '<i class="far fa-star"></i>';
                                          }
                                      }
                                      ?>
                                  </div>
                                  <span class="rating-count">(<?php echo htmlspecialchars($averageRating); ?>)</span>
                              </div>
                         </div>
                     </div>
                     <?php endforeach; ?>
                <?php endif; ?>
            </div>

            <!-- Action Buttons -->
            <div class="action-buttons">
                <button class="edit-all-btn" title="Edit All Movies">
                    <i class="fas fa-edit"></i>
                </button>
                <a href="upload.php" class="upload-btn" title="Upload New Movie">
                    <i class="fas fa-plus"></i>
                </a>
            </div>
        </main>
    </div>
    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
    <script>
         // Add specific JS for manage page actions (edit/delete) if needed via AJAX
         document.addEventListener('DOMContentLoaded', function() {
             // Example delete button functionality (requires backend endpoint)
             document.querySelectorAll('.delete-btn').forEach(button => {
                 button.addEventListener('click', function(e) {
                     e.stopPropagation(); // Prevent parent click
                     e.preventDefault();

                     const movieId = this.closest('.movie-card').getAttribute('data-movie-id');
                     if (confirm('Are you sure you want to delete this movie?')) {
                         // TODO: Send AJAX request to delete movie endpoint
                         console.log("Deleting movie ID:", movieId);
                         // Example AJAX call (endpoint needs to be created)
                         /*
                         fetch('delete_movie.php', { // Create this PHP file
                             method: 'POST',
                             headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                             body: `movie_id=${movieId}`
                         })
                         .then(response => response.json())
                         .then(data => {
                             if (data.success) {
                                 alert('Movie deleted successfully!');
                                 this.closest('.movie-card').remove(); // Remove card from DOM
                                 // Check if grid is now empty and show empty state
                                  const grid = document.querySelector('.movies-grid');
                                   if (grid && grid.children.length === 0) {
                                       if (!document.querySelector('.empty-state:not(.search-empty-state)')) { // Check for default empty state
                                            const emptyState = document.createElement('div');
                                            emptyState.className = 'empty-state';
                                            emptyState.innerHTML = `
                                               <i class="fas fa-film"></i>
                                               <p>No movies uploaded yet</p>
                                               <p class="subtitle">Start adding your movies by clicking the upload button below</p>
                                           `; // Customize message
                                           grid.appendChild(emptyState);
                                       }
                                   }

                             } else {
                                 alert('Failed to delete movie: ' + data.message);
                             }
                         })
                         .catch(error => {
                             console.error('Error deleting movie:', error);
                             alert('An error occurred during deletion.');
                         });
                         */
                         alert("Delete functionality not yet implemented."); // Placeholder

                     }
                 });
             });

              // Example edit button functionality (requires dedicated edit page or modal)
             document.querySelectorAll('.edit-btn').forEach(button => {
                 button.addEventListener('click', function(e) {
                      e.stopPropagation();
                      e.preventDefault();
                      const movieId = this.closest('.movie-card').getAttribute('data-movie-id');
                      // TODO: Redirect to an edit page or open an edit modal with movie ID
                      console.log("Editing movie ID:", movieId);
                      alert("Edit functionality not yet implemented."); // Placeholder
                 });
             });
         });
    </script>
</body>
</html>

```

**14. `manage/upload.php` (Modifikasi)**

*   Ubah path CSS (`../css/style.css`).
*   Sertakan `config/database.php` dan `includes/auth_helper.php`.
*   Tambahkan logika untuk menangani form POST, upload file, dan menyimpan ke database.
*   Perbaiki path link sidebar.

```php
<?php
// manage/upload.php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php';

// Redirect to login if not logged in
require_login();

$userId = get_user_id();
$error_message = null;
$success_message = null;

// Handle POST request for form submission
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // Get form data
    $title = trim($_POST['movie-title'] ?? '');
    $summary = trim($_POST['movie-summary'] ?? '');
    $release_date = $_POST['release-date'] ?? '';
    $duration_hours = intval($_POST['duration-hours'] ?? 0);
    $duration_minutes = intval($_POST['duration-minutes'] ?? 0);
    $age_rating = $_POST['age-rating'] ?? '';
    $genres = $_POST['genre'] ?? []; // Array of selected genres
    $trailer_link = trim($_POST['trailer-link'] ?? '');
    $uploaded_by = $userId; // User ID from session

    // File upload handling
    $poster_image_path = null;
    $trailer_file_path = null;

    // --- Validation ---
    $errors = [];
    if (empty($title)) $errors[] = 'Movie title is required.';
    if (empty($summary)) $errors[] = 'Movie summary is required.';
    if (empty($release_date)) $errors[] = 'Release date is required.';
    if ($duration_hours < 0 || $duration_minutes < 0 || $duration_minutes > 59) $errors[] = 'Invalid duration.';
    if (empty($age_rating)) $errors[] = 'Age rating is required.';
    // Genres are optional based on form design, but you could require at least one
    if (empty($_FILES['movie-poster']['name']) && empty($poster_image_path)) {
        $errors[] = 'Movie poster is required.';
    }
    // Trailer link or file is optional
    if (empty($trailer_link) && empty($_FILES['trailer-file']['name'])) {
        // No trailer provided, this is allowed
    }

     // Basic file upload validation for poster
     if (!empty($_FILES['movie-poster']['name'])) {
         $poster = $_FILES['movie-poster'];
         $allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
         $maxSize = 5 * 1024 * 1024; // 5MB

         if ($poster['error'] !== UPLOAD_ERR_OK) {
             $errors[] = 'Poster upload error: ' . $poster['error'];
         } elseif (!in_array($poster['type'], $allowedTypes)) {
             $errors[] = 'Invalid poster file type. Only JPG, PNG, GIF, WEBP allowed.';
         } elseif ($poster['size'] > $maxSize) {
             $errors[] = 'Poster file size exceeds limit (5MB).';
         }
     }

     // Basic file upload validation for trailer file if uploaded
     if (!empty($_FILES['trailer-file']['name'])) {
          if (!empty($trailer_link)) {
               // Both provided, might want to error or prioritize one
               $errors[] = 'Please provide either a trailer link OR upload a trailer file, not both.';
          } else {
               $trailerFile = $_FILES['trailer-file'];
               $allowedTypes = ['video/mp4', 'video/webm', 'video/ogg']; // Example video types
               $maxSize = 50 * 1024 * 1024; // 50MB

               if ($trailerFile['error'] !== UPLOAD_ERR_OK) {
                   $errors[] = 'Trailer file upload error: ' . $trailerFile['error'];
               } elseif (!in_array($trailerFile['type'], $allowedTypes)) {
                   $errors[] = 'Invalid trailer file type. Only MP4, WebM, Ogg allowed.';
               } elseif ($trailerFile['size'] > $maxSize) {
                   $errors[] = 'Trailer file size exceeds limit (50MB).';
               }
          }
     }


    // If there are validation errors, display them
    if (!empty($errors)) {
        $error_message = implode('<br>', $errors);
    } else {
        // --- Handle File Uploads (Poster) ---
        if (!empty($_FILES['movie-poster']['tmp_name']) && $_FILES['movie-poster']['error'] === UPLOAD_ERR_OK) {
            $uploadDir = __DIR__ . '/../uploads/posters/'; // Adjusted path
            if (!is_dir($uploadDir)) {
                mkdir($uploadDir, 0777, true); // Create directory if it doesn't exist
            }
            $fileName = uniqid() . '_' . basename($_FILES['movie-poster']['name']);
            $targetFilePath = $uploadDir . $fileName;

            if (move_uploaded_file($_FILES['movie-poster']['tmp_name'], $targetFilePath)) {
                $poster_image_path = '/uploads/posters/' . $fileName; // Path relative to webroot
            } else {
                $errors[] = 'Failed to move uploaded poster file.';
            }
        }

        // --- Handle File Uploads (Trailer File) ---
        if (!empty($_FILES['trailer-file']['tmp_name']) && $_FILES['trailer-file']['error'] === UPLOAD_ERR_OK && empty($errors)) { // Only proceed if no poster errors
             $uploadDir = __DIR__ . '/../uploads/trailers/'; // Adjusted path
             if (!is_dir($uploadDir)) {
                 mkdir($uploadDir, 0777, true);
             }
             $fileName = uniqid() . '_' . basename($_FILES['trailer-file']['name']);
             $targetFilePath = $uploadDir . $fileName;

             if (move_uploaded_file($_FILES['trailer-file']['tmp_name'], $targetFilePath)) {
                 $trailer_file_path = '/uploads/trailers/' . $fileName; // Path relative to webroot
                 $trailer_link = null; // Ensure link is null if file uploaded
             } else {
                 $errors[] = 'Failed to move uploaded trailer file.';
             }
        } else {
             // If no trailer file uploaded, use the link if provided
             $trailer_file_path = null;
             // $trailer_link is already set from $_POST
        }


        // If no errors after uploads, insert into database
        if (empty($errors)) {
            try {
                $movieId = createMovie(
                    $title,
                    $summary,
                    $release_date,
                    $duration_hours,
                    $duration_minutes,
                    $age_rating,
                    $poster_image_path, // Use uploaded path
                    $trailer_link,    // Use link
                    $trailer_file_path, // Use file path
                    $uploaded_by
                );

                if ($movieId) {
                    // Add genres
                    if (!empty($genres)) {
                        foreach ($genres as $genre) {
                             // Validate genre against allowed ENUM values in database.php if not already done
                             addMovieGenre($movieId, $genre); // Function handles potential duplicates
                        }
                    }

                    $success_message = "Movie '$title' uploaded successfully!";
                    // Redirect to manage page after successful upload
                    header('Location: index.php');
                    exit;

                } else {
                    $errors[] = 'Failed to create movie entry in database.';
                }

            } catch (PDOException $e) {
                error_log("Database error during movie upload: " . $e->getMessage());
                $errors[] = 'Terjadi kesalahan internal saat menyimpan film. Silakan coba lagi.';
            }
        }

        // If there are still errors (e.g., from uploads or DB insert), display them
        if (!empty($errors)) {
            $error_message = implode('<br>', $errors);
            // Clean up uploaded files if DB insert failed (optional but good practice)
            if ($poster_image_path && file_exists(__DIR__ . '/../' . $poster_image_path)) {
                 unlink(__DIR__ . '/../' . $poster_image_path);
            }
             if ($trailer_file_path && file_exists(__DIR__ . '/../' . $trailer_file_path)) {
                 unlink(__DIR__ . '/../' . $trailer_file_path);
             }
        }
    }
}

?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Upload Movie - RatingTales</title>
     <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
</head>
<body>
    <div class="container">
        <!-- Sidebar Navigation -->
        <nav class="sidebar">
            <h2 class="logo">RATE-TALES</h2>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li> <!-- Adjusted path -->
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favorites</span></a></li> <!-- Adjusted path -->
                <li><a href="../review/index.php"><i class="fas fa-star"></i> <span>Review</span></a></li> <!-- Adjusted path -->
                <li><a href="index.php" class="active"><i class="fas fa-film"></i> <span>Manage</span></a></li> <!-- Adjusted path -->
            </ul>
            <ul class="bottom-links">
                <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Adjusted path -->
            </ul>
        </nav>

        <!-- Main Content -->
        <main class="main-content">
            <header>
                 <!-- Header content if any -->
            </header>

            <div class="upload-container">
                <h1>Upload New Movie</h1>

                 <?php if ($error_message): ?>
                     <div class="error-message" style="margin-bottom: 20px;"><?php echo htmlspecialchars($error_message); ?></div>
                 <?php endif; ?>
                 <?php if ($success_message): ?>
                     <div class="success-message" style="margin-bottom: 20px;"><?php echo htmlspecialchars($success_message); ?></div>
                 <?php endif; ?>

                <form class="upload-form" action="upload.php" method="post" enctype="multipart/form-data">
                    <div class="form-layout">
                        <div class="form-main">
                            <div class="form-group">
                                <label for="movie-title">Movie Title</label>
                                <input type="text" id="movie-title" name="movie-title" required value="<?php echo htmlspecialchars($_POST['movie-title'] ?? ''); ?>">
                            </div>

                            <div class="form-group">
                                <label for="movie-summary">Movie Summary</label>
                                <textarea id="movie-summary" name="movie-summary" rows="4" required><?php echo htmlspecialchars($_POST['movie-summary'] ?? ''); ?></textarea>
                            </div>

                            <div class="form-group">
                                <label>Genre</label>
                                <div class="genre-options">
                                    <?php
                                    // Dynamically generate genre options based on ENUM values from DB if possible,
                                    // or list them manually matching the ENUM ('action', 'adventure', 'comedy', 'drama', 'horror')
                                    $genres = ['action', 'adventure', 'comedy', 'drama', 'horror'];
                                    $selectedGenres = $_POST['genre'] ?? []; // Get previously selected genres on error
                                    foreach ($genres as $genreOption):
                                        $isChecked = in_array($genreOption, $selectedGenres);
                                    ?>
                                    <label class="checkbox-label">
                                        <input type="checkbox" name="genre[]" value="<?php echo $genreOption; ?>" <?php echo $isChecked ? 'checked' : ''; ?>>
                                        <span><?php echo ucfirst($genreOption); ?></span> <!-- Display capitalized -->
                                    </label>
                                    <?php endforeach; ?>
                                </div>
                            </div>

                            <div class="form-group">
                                <label for="age-rating">Age Rating</label>
                                <?php
                                // Options should match the ENUM in the DB: ('G', 'PG', 'PG-13', 'R', 'NC-17')
                                $ageRatings = [
                                    'G' => 'G (General Audience)',
                                    'PG' => 'PG (Parental Guidance)',
                                    'PG-13' => 'PG-13 (Parental Guidance for children under 13)',
                                    'R' => 'R (Restricted)',
                                    'NC-17' => 'NC-17 (Adults Only)'
                                ];
                                $selectedAgeRating = $_POST['age-rating'] ?? '';
                                ?>
                                <select id="age-rating" name="age-rating" required>
                                    <option value="">Select age rating</option>
                                    <?php foreach ($ageRatings as $value => $label): ?>
                                         <option value="<?php echo $value; ?>" <?php echo ($selectedAgeRating === $value) ? 'selected' : ''; ?>>
                                              <?php echo $label; ?>
                                         </option>
                                    <?php endforeach; ?>
                                </select>
                            </div>

                            <div class="form-group">
                                <label for="movie-trailer">Movie Trailer</label>
                                <div class="trailer-input">
                                    <input type="text" id="trailer-link" name="trailer-link" placeholder="Enter YouTube video URL" value="<?php echo htmlspecialchars($_POST['trailer-link'] ?? ''); ?>">
                                    <span class="trailer-note">* Paste YouTube video URL</span>
                                </div>
                                <div class="trailer-upload">
                                    <input type="file" id="trailer-file" name="trailer-file" accept="video/*">
                                    <span class="trailer-note">* Or upload video file</span>
                                </div>
                            </div>
                        </div>

                        <div class="form-side">
                            <div class="poster-upload form-group"> <!-- Add form-group class for consistent spacing -->
                                <label for="movie-poster">Movie Poster</label>
                                <div class="upload-area" id="upload-area">
                                    <i class="fas fa-image"></i>
                                    <p>Click or drag image here</p>
                                    <input type="file" id="movie-poster" name="movie-poster" accept="image/*" required>
                                </div>
                                <!-- Optional: Display preview after file selection -->
                                 <img id="poster-preview" src="#" alt="Poster Preview" style="display: none; max-width: 100%; height: auto; margin-top: 10px; border-radius: 8px;">
                            </div>

                            <div class="advanced-settings form-section"> <!-- Add form-section class for spacing -->
                                <h3>Advanced Settings</h3>
                                <div class="form-group">
                                    <label for="release-date">Release Date</label>
                                    <input type="date" id="release-date" name="release-date" required value="<?php echo htmlspecialchars($_POST['release-date'] ?? ''); ?>">
                                </div>

                                <div class="form-group">
                                    <label for="duration-hours">Film Duration</label>
                                    <div class="duration-inputs">
                                        <div class="duration-field">
                                            <input type="number" id="duration-hours" name="duration-hours" min="0" placeholder="Hours" required value="<?php echo htmlspecialchars($_POST['duration-hours'] ?? ''); ?>">
                                            <span>Hours</span>
                                        </div>
                                        <div class="duration-field">
                                            <input type="number" id="duration-minutes" name="duration-minutes" min="0" max="59" placeholder="Minutes" required value="<?php echo htmlspecialchars($_POST['duration-minutes'] ?? ''); ?>">
                                            <span>Minutes</span>
                                        </div>
                                    </div>
                                </div>
                                <!-- Add other advanced settings here if needed -->
                            </div>
                        </div>
                    </div>

                    <div class="form-actions">
                        <button type="button" class="cancel-btn" onclick="window.location.href='index.php'">Cancel</button>
                        <button type="submit" class="submit-btn">Upload Movie</button>
                    </div>
                </form>
            </div>
        </main>
    </div>
     <script>
         // Script for file preview and optional client-side validation hints
         document.addEventListener('DOMContentLoaded', function() {
             const posterInput = document.getElementById('movie-poster');
             const posterPreview = document.getElementById('poster-preview');
             const uploadArea = document.getElementById('upload-area');
             const trailerLinkInput = document.getElementById('trailer-link');
             const trailerFileInput = document.getElementById('trailer-file');

             // Poster Preview
             if (posterInput && posterPreview && uploadArea) {
                 posterInput.addEventListener('change', function(event) {
                     const file = event.target.files[0];
                     if (file) {
                         const reader = new FileReader();
                         reader.onload = function(e) {
                             posterPreview.src = e.target.result;
                             posterPreview.style.display = 'block';
                             uploadArea.style.display = 'none'; // Hide upload area
                         }
                         reader.readAsDataURL(file);
                     } else {
                         posterPreview.src = '#';
                         posterPreview.style.display = 'none';
                         uploadArea.style.display = 'flex'; // Show upload area
                     }
                 });

                 // Optional: Handle drag and drop for poster
                 uploadArea.addEventListener('dragover', (event) => {
                     event.preventDefault();
                     uploadArea.classList.add('drag-over');
                 });
                 uploadArea.addEventListener('dragleave', (event) => {
                      event.preventDefault();
                     uploadArea.classList.remove('drag-over');
                 });
                 uploadArea.addEventListener('drop', (event) => {
                     event.preventDefault();
                     uploadArea.classList.remove('drag-over');
                     const files = event.dataTransfer.files;
                     if (files.length > 0) {
                         posterInput.files = files; // Assign dropped files to input
                         const changeEvent = new Event('change');
                         posterInput.dispatchEvent(changeEvent); // Trigger change event for preview
                     }
                 });
             }

             // Trailer link vs File behavior (client-side indication)
             if (trailerLinkInput && trailerFileInput) {
                 trailerLinkInput.addEventListener('input', function() {
                     if (this.value) {
                         trailerFileInput.disabled = true;
                         // Optional: Add a visual hint that file upload is disabled
                         trailerFileInput.closest('.trailer-upload').style.opacity = 0.5;
                     } else {
                         trailerFileInput.disabled = false;
                         trailerFileInput.closest('.trailer-upload').style.opacity = 1;
                     }
                 });

                 trailerFileInput.addEventListener('change', function() {
                     if (this.files.length > 0) {
                         trailerLinkInput.disabled = true;
                         // Optional: Add a visual hint that link input is disabled
                         trailerLinkInput.closest('.trailer-input').style.opacity = 0.5;
                     } else {
                         trailerLinkInput.disabled = false;
                         trailerLinkInput.closest('.trailer-input').style.opacity = 1;
                     }
                 });

                 // Initial state check in case form was submitted with error and values were restored
                 if (trailerLinkInput.value) {
                     trailerFileInput.disabled = true;
                      trailerFileInput.closest('.trailer-upload').style.opacity = 0.5;
                 } else if (trailerFileInput.files.length > 0 || (trailerFileInput.value && trailerFileInput.value !== '')) { // Check if a file was selected (even if value string is tricky)
                      trailerLinkInput.disabled = true;
                       trailerLinkInput.closest('.trailer-input').style.opacity = 0.5;
                 }
             }

         });
     </script>
</body>
</html>

```

**15. `acc_page/index.php` (Modifikasi)**

*   Ubah path CSS (`../css/style.css`).
*   Ubah path JS (`../js/main.js`).
*   Sertakan `config/database.php` dan `includes/auth_helper.php`.
*   Ambil data pengguna yang login dari database.
*   Hapus bagian "My Post" karena tidak didukung skema DB.
*   Implementasikan fungsi edit profil (perlu AJAX endpoint).
*   Perbaiki path link sidebar.

```php
<?php
// acc_page/index.php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php';

// Redirect to login if not logged in
require_login();

$userId = get_user_id();
$user = getUserById($userId);

// Handle case where user somehow isn't found (shouldn't happen after require_login)
if (!$user) {
     // Log error
     error_log("Logged in user ID {$userId} not found in database.");
     // Redirect to logout or error page
     header('Location: ../autentikasi/logout.php');
     exit;
}

// Assuming profile_image is stored as a path relative to webroot /uploads/profiles/
$profileImagePath = $user['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($user['username'] ?? 'User') . '&background=random';

// Bio placeholder text
$bioText = $user['bio'] ? htmlspecialchars($user['bio']) : 'Click to add bio...';

// --- Note: "My Posts" section is removed as it's not supported by the provided DB schema. ---

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Profile</title>
    <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">
</head>
<body class="profile-page"> <!-- Add body class -->
    <div class="container">
        <nav class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li> <!-- Adjusted path -->
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favorites</span></a></li> <!-- Adjusted path -->
                <li><a href="../review/index.php"><i class="fas fa-star"></i> <span>Review</span></a></li> <!-- Adjusted path -->
                <li><a href="../manage/index.php"><i class="fas fa-film"></i> <span>Manage</span></a></li> <!-- Adjusted path -->
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Adjusted path -->
                </ul>
            </div>
        </nav>
        <main class="main-content">
            <div class="profile-header">
                <div class="profile-info">
                    <div class="profile-image">
                        <img src="<?php echo htmlspecialchars($profileImagePath); ?>" alt="Profile Picture">
                        <!-- Optional: Add an icon/button to change profile picture -->
                         <!-- <i class="fas fa-camera change-photo-icon"></i> -->
                    </div>
                    <div class="profile-details">
                        <h1>
                            <!-- Display Name (Full Name from DB) -->
                            <span id="displayName"><?php echo htmlspecialchars($user['full_name']); ?></span>
                            <i class="fas fa-pen edit-icon" onclick="toggleEdit('full_name')"></i> <!-- Use DB field name 'full_name' -->
                        </h1>
                        <p class="username">
                            @<span id="username"><?php echo htmlspecialchars($user['username']); ?></span>
                            <i class="fas fa-pen edit-icon" onclick="toggleEdit('username')"></i> <!-- Use DB field name 'username' -->
                        </p>
                         <!-- Optional: Display Email, Age, Gender -->
                         <!-- <p>Email: <?php echo htmlspecialchars($user['email']); ?></p> -->
                         <!-- <p>Member Since: <?php echo date('M Y', strtotime($user['created_at'])); ?></p> -->
                    </div>
                </div>
                <div class="about-me">
                    <h2>ABOUT ME:</h2>
                    <div class="about-content" id="bio" onclick="toggleEdit('bio')"> <!-- Use DB field name 'bio' -->
                        <?php echo $bioText; ?> <!-- Bio text is already HTML escaped -->
                    </div>
                </div>
            </div>

             <!-- Removed "My Posts" section as it's not in the database schema -->
            <!--
            <div class="posts-section">
                <h2>My Post</h2>
                <div class="posts-grid">
                     ... posts loop ...
                </div>
            </div>
            -->
        </main>
    </div>
    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
    <script>
        // Note: toggleEdit function is now in js/main.js
        // It needs to send AJAX calls to acc_page/update_profile.php

        // Example implementation for profile picture change (requires server endpoint)
        /*
        document.addEventListener('DOMContentLoaded', function() {
             const changePhotoIcon = document.querySelector('.change-photo-icon');
             const profileImage = document.querySelector('.profile-image img');

             if (changePhotoIcon && profileImage) {
                 changePhotoIcon.addEventListener('click', function() {
                     // Trigger file input click or open a modal
                     alert("Profile picture change not yet implemented."); // Placeholder
                 });
             }
        });
        */
    </script>
</body>
</html>

```

**16. `review/index.php` (Modifikasi)**

*   Ubah path CSS (`../css/style.css`).
*   Ubah path JS (`../js/main.js`).
*   Sertakan `config/database.php` dan `includes/auth_helper.php`.
*   Ambil *semua* film dari database menggunakan `getAllMovies()`.
*   Ambil rata-rata rating untuk setiap film menggunakan `getAverageMovieRating()`.
*   Update `showMovieDetailsPopup` JS function to pass movie ID and fetch/display correct data in the popup.
*   Link cards to open the popup.
*   Implementasikan action button "Favorite" (AJAX endpoint).
*   Perbaiki path link sidebar.

```php
<?php
// review/index.php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php';

// Redirect to login if not logged in (Optional: make review page public?)
// Let's make it require login for consistency with other user pages
require_login();

// Get logged-in user ID for favorite check
$userId = get_user_id();

// Fetch all movies from the database
$movies = getAllMovies();

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Review</title>
     <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">
</head>
<body class="review-page"> <!-- Add body class for JS targeting -->
    <div class="container">
        <nav class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li> <!-- Adjusted path -->
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favorites</span></a></li> <!-- Adjusted path -->
                <li class="active"><a href="#"><i class="fas fa-star"></i> <span>Review</span></a></li>
                <li><a href="../manage/index.php"><i class="fas fa-film"></i> <span>Manage</span></a></li> <!-- Adjusted path -->
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Adjusted path -->
                </ul>
            </div>
        </nav>
        <main class="main-content">
            <div class="review-header">
                <h1>Movie Reviews</h1>
                <div class="search-bar">
                    <input type="text" placeholder="Search movies...">
                    <button><i class="fas fa-search"></i></button>
                </div>
            </div>
            <div class="review-grid">
                <?php if (!empty($movies)): ?>
                    <?php foreach ($movies as $movie):
                        $releaseYear = date('Y', strtotime($movie['release_date']));
                        $genresString = !empty($movie['genres']) ? implode(', ', $movie['genres']) : 'N/A';
                        $averageRating = getAverageMovieRating($movie['movie_id']); // Fetch average rating
                        $isFavorited = isFavorite($movie['movie_id'], $userId); // Check if favorite
                    ?>
                    <div class="movie-card" data-movie-id="<?php echo $movie['movie_id']; ?>"
                         onclick="showMovieDetailsPopup(
                             <?php echo $movie['movie_id']; ?>,
                             '<?php echo htmlspecialchars($movie['title'], ENT_QUOTES, 'UTF-8'); ?>',
                             '<?php echo htmlspecialchars($releaseYear, ENT_QUOTES, 'UTF-8'); ?>',
                             '<?php echo htmlspecialchars($genresString, ENT_QUOTES, 'UTF-8'); ?>',
                             '<?php echo htmlspecialchars(json_encode($movie['summary']), ENT_QUOTES, 'UTF-8'); ?>', <!-- Pass summary as JSON string -->
                             '<?php echo htmlspecialchars($averageRating, ENT_QUOTES, 'UTF-8'); ?>'
                         )">
                        <div class="movie-poster">
                            <img src="<?php echo htmlspecialchars($movie['poster_image']); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                            <div class="movie-actions">
                                <!-- Favorite Button (updated with classes for JS targeting) -->
                                <button class="action-btn favorite-btn <?php echo $isFavorited ? 'fas' : 'far'; ?> fa-heart" title="<?php echo $isFavorited ? 'Remove from favorites' : 'Add to favorites'; ?>"></button>
                                <!-- Optional: Quick Review Button -->
                                <!-- <button class="action-btn review-btn" title="Write a quick review"><i class="fas fa-pen"></i></button> -->
                            </div>
                        </div>
                        <div class="movie-details">
                            <h3><?php echo htmlspecialchars($movie['title']); ?></h3>
                            <p class="movie-info"><?php echo htmlspecialchars($releaseYear); ?> | <?php echo htmlspecialchars($genresString); ?></p>
                            <div class="rating">
                                <div class="stars">
                                    <?php
                                    $rating = floatval($averageRating); // Use fetched average rating
                                    for ($i = 1; $i <= 5; $i++) {
                                        if ($i <= $rating) {
                                            echo '<i class="fas fa-star"></i>';
                                        } else if ($i - 0.5 <= $rating) {
                                            echo '<i class="fas fa-star-half-alt"></i>';
                                        } else {
                                            echo '<i class="far fa-star"></i>';
                                        }
                                    }
                                    ?>
                                </div>
                                <span class="rating-count">(<?php echo htmlspecialchars($averageRating); ?>)</span>
                            </div>
                        </div>
                    </div>
                    <?php endforeach; ?>
                <?php else: ?>
                     <div class="empty-state">
                        <i class="fas fa-film"></i>
                        <p>No movies available yet</p>
                        <p class="subtitle">Movies will appear here when they are uploaded</p>
                    </div>
                <?php endif; ?>
            </div>
        </main>
    </div>

    <!-- Movie Details Popup -->
    <div id="movie-details-popup" class="movie-details-popup">
        <div class="popup-content">
            <div class="popup-header">
                <h2 id="popup-title"></h2>
                <div class="popup-rating">
                     <!-- Stars for user rating input -->
                    <div class="rating-stars">
                        <!-- Stars will be dynamically added by JS -->
                         <i class="far fa-star" data-rating="1"></i>
                         <i class="far fa-star" data-rating="2"></i>
                         <i class="far fa-star" data-rating="3"></i>
                         <i class="far fa-star" data-rating="4"></i>
                         <i class="far fa-star" data-rating="5"></i>
                    </div>
                    <!-- Average rating display -->
                    <span id="popup-rating"></span>
                </div>
            </div>
            <p id="popup-year-genre"></p>
            <p id="popup-description"></p>
            <div class="popup-actions">
                 <!-- Note: These actions might be better handled by the main JS `action-btn` listeners -->
                 <!-- Updated classes for consistency -->
                 <button class="action-button add-favorite-popup-btn"><i class="fas fa-heart"></i> Add to Favorites</button>
                 <button class="action-button read-more-btn" onclick="openMovieDetails()"><i class="fas fa-book-open"></i> Read More</button>
            </div>
        </div>
    </div>

    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
    <script>
        // Pass movie ID to popup action button for consistency
        // The `showMovieDetailsPopup` function in main.js sets `currentMovieData.id`.
        // The `openMovieDetails` function in main.js uses `currentMovieData.id` to redirect.

         // Add event listener for the favorite button *inside the popup* if it exists
         const popupFavoriteBtn = document.querySelector('.popup-content .add-favorite-popup-btn');
         if (popupFavoriteBtn) {
              popupFavoriteBtn.addEventListener('click', function(e) {
                   e.stopPropagation(); // Prevent popup close
                   e.preventDefault();

                   if (currentMovieData && currentMovieData.id) {
                        // Find the corresponding card button to check its state
                        const cardButton = document.querySelector(`.movie-card[data-movie-id="${currentMovieData.id}"] .favorite-btn`);
                        const isFavorited = cardButton ? cardButton.classList.contains('fas') : false;

                        fetch('toggle_favorite.php', { // Adjust path to favorite/toggle_favorite.php
                            method: 'POST',
                            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                            body: `movie_id=${currentMovieData.id}&action=${isFavorited ? 'remove' : 'add'}`
                        })
                        .then(response => response.json())
                        .then(data => {
                            if (data.success) {
                                 if (cardButton) { // Update the icon on the original card
                                     if (data.action === 'added') {
                                         cardButton.classList.remove('far');
                                         cardButton.classList.add('fas');
                                          cardButton.title = 'Remove from favorites';
                                          popupFavoriteBtn.innerHTML = '<i class="fas fa-heart"></i> Remove from Favorites'; // Update popup button text/icon
                                     } else { // Removed
                                         cardButton.classList.remove('fas');
                                         cardButton.classList.add('far');
                                         cardButton.title = 'Add to favorites';
                                          popupFavoriteBtn.innerHTML = '<i class="fas fa-heart"></i> Add to Favorites'; // Update popup button text/icon
                                     }
                                 }
                                // alert(data.message); // User feedback
                            } else {
                                console.error("Failed to toggle favorite:", data.message);
                                alert("Failed to update favorites: " + data.message); // User feedback
                            }
                        })
                        .catch(error => {
                            console.error("AJAX Error toggling favorite:", error);
                            alert("An error occurred while updating favorites."); // Generic error
                        });
                   } else {
                        console.error("Movie ID missing for favorite toggle in popup.");
                        alert("Could not add to favorites.");
                   }
              });
         }


    </script>
</body>
</html>
```

**17. `review/movie-details.php` (Modifikasi)**

*   Ubah path CSS (`../css/style.css`).
*   Ubah path JS (`../js/main.js`).
*   Sertakan `config/database.php` dan `includes/auth_helper.php`.
*   Ambil movie ID dari URL (`$_GET['id']`).
*   Ambil detail film menggunakan `getMovieById()`.
*   Ambil komentar menggunakan `getMovieReviews()`.
*   Tampilkan data dinamis dari DB.
*   Implementasikan form submit komentar (perlu AJAX endpoint).
*   Implementasikan action button "Add to Favorites" (AJAX endpoint).
*   Implementasikan action button "Watch Trailer" (menggunakan data trailer dari DB).
*   Hapus inline `<style>`.
*   Perbaiki path link sidebar.

```php
<?php
// review/movie-details.php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php';

// Redirect to login if not logged in (Optional: make details page public?)
// Let's make it require login
require_login();

$userId = get_user_id(); // Get logged-in user ID

// Get movie ID from URL
$movieId = $_GET['id'] ?? null;

// Fetch movie details from database
$movie = null;
$reviews = [];
$averageRating = '0.0';
$isFavorited = false;
$trailerUrlToUse = null; // Determine which trailer link/file to use

if ($movieId) {
    $movie = getMovieById($movieId);
    if ($movie) {
        $reviews = getMovieReviews($movieId); // Fetch reviews for this movie
        $averageRating = getAverageMovieRating($movieId); // Fetch average rating
        $isFavorited = isFavorite($movieId, $userId); // Check if favorite

        // Determine which trailer to use: trailer_link (YouTube) or trailer_file (local)
        if (!empty($movie['trailer_link'])) {
            $trailerUrlToUse = htmlspecialchars($movie['trailer_link']);
        } elseif (!empty($movie['trailer_file'])) {
             // For local file, the player logic in JS needs adjustment (e.g. use a <video> tag)
             // For now, we'll just store the path. The JS playTrailer needs modification.
             // Let's prioritize the link for the current JS implementation
             $trailerUrlToUse = htmlspecialchars($movie['trailer_file']); // Still pass the path, JS will need to handle it
             // ALERT: Current JS playTrailer only handles YouTube embed logic well.
             // Handling local files requires different JS (creating <video> tag).
             // For this scope, we'll pass the file path but note the JS limitation.
             // A simple approach for file: just link to it or open in a new tab
             // For now, let's pass it, JS will need to distinguish.
        }

    }
}

// Handle movie not found
if (!$movie) {
    // Display an error or redirect
    header('Location: index.php'); // Redirect back to review list
    exit;
}

// --- Handle Comment Submission (AJAX endpoint is better, but basic form submit here) ---
// If you want form submit here, uncomment and implement POST handling.
// For AJAX, create review/submit_review.php and handle the POST there.
/*
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['comment_text'])) {
     $commentText = trim($_POST['comment_text']);
     $ratingSubmitted = $_POST['rating_value'] ?? null; // If rating is submitted with comment

     if (!empty($commentText)) {
          // TODO: Validate ratingSubmitted if present
          // TODO: Call createReview($movieId, $userId, $ratingSubmitted, $commentText);
          // TODO: Redirect back or refresh page to see comment
          // Example: createReview($movieId, $userId, 4.0, $commentText); // Use a default rating or get from form
          // header("Location: movie-details.php?id={$movieId}");
          // exit;
     }
     // Handle empty comment or errors
}
*/

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?php echo htmlspecialchars($movie['title']); ?> - RATE-TALES</title>
     <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">
    <!-- No inline styles needed anymore, moved to css/style.css -->
</head>
<body>
    <div class="container">
        <nav class="sidebar">
            <div class="logo">
                <h2>RATE-TALES</h2>
            </div>
            <ul class="nav-links">
                <li><a href="../beranda/index.php"><i class="fas fa-home"></i> <span>Home</span></a></li> <!-- Adjusted path -->
                <li><a href="../favorite/index.php"><i class="fas fa-heart"></i> <span>Favorites</span></a></li> <!-- Adjusted path -->
                <li class="active"><a href="index.php"><i class="fas fa-star"></i> <span>Review</span></a></li> <!-- Link back to review list -->
                <li><a href="../manage/index.php"><i class="fas fa-film"></i> <span>Manage</span></a></li> <!-- Adjusted path -->
            </ul>
            <div class="bottom-links">
                <ul>
                    <li><a href="../autentikasi/logout.php"><i class="fas fa-sign-out-alt"></i> <span>Logout</span></a></li> <!-- Adjusted path -->
                </ul>
            </div>
        </nav>
        <main class="main-content">
            <div class="movie-details-page">
                <a href="index.php" class="back-button"> <!-- Link back to review list -->
                    <i class="fas fa-arrow-left"></i>
                    <span>Back to Reviews</span>
                </a>
                <div class="movie-header">
                    <div class="movie-poster-large">
                        <img id="movie-poster" src="<?php echo htmlspecialchars($movie['poster_image']); ?>" alt="<?php echo htmlspecialchars($movie['title']); ?>">
                    </div>
                    <div class="movie-info-large">
                        <h1 id="movie-title" class="movie-title-large"><?php echo htmlspecialchars($movie['title']); ?></h1>
                        <p id="movie-meta" class="movie-meta">
                            <?php echo htmlspecialchars(date('Y', strtotime($movie['release_date']))); ?> |
                            <?php echo htmlspecialchars(!empty($movie['genres']) ? implode(', ', $movie['genres']) : 'N/A'); ?> |
                            <?php echo htmlspecialchars($movie['duration_hours'] . 'h ' . $movie['duration_minutes'] . 'm'); ?> |
                            Rated: <?php echo htmlspecialchars($movie['age_rating']); ?>
                        </p>
                        <div class="rating-large">
                             <!-- Display average rating using stars -->
                            <div class="stars">
                                <?php
                                 $rating = floatval($averageRating);
                                 for ($i = 1; $i <= 5; $i++) {
                                     if ($i <= $rating) {
                                         echo '<i class="fas fa-star"></i>';
                                     } else if ($i - 0.5 <= $rating) {
                                         echo '<i class="fas fa-star-half-alt"></i>';
                                     } else {
                                         echo '<i class="far fa-star"></i>';
                                     }
                                 }
                                ?>
                            </div>
                            <span id="movie-rating"><?php echo htmlspecialchars($averageRating); ?>/5</span>
                        </div>
                        <p id="movie-description" class="movie-description"><?php echo htmlspecialchars($movie['summary']); ?></p>
                        <div class="action-buttons">
                            <button class="action-button watch-trailer" onclick="playTrailer('<?php echo $trailerUrlToUse; ?>')">
                                <i class="fas fa-play"></i>
                                <span>Watch Trailer</span>
                            </button>
                            <!-- Favorite Button (updated with classes for JS targeting) -->
                            <button class="action-button favorite-btn" data-movie-id="<?php echo $movie['movie_id']; ?>">
                                <i class="fas <?php echo $isFavorited ? 'fa-heart' : 'fa-heart'; ?>"></i> <!-- Use fas or far based on state if different icons needed -->
                                <span><?php echo $isFavorited ? 'Remove from Favorites' : 'Add to Favorites'; ?></span>
                            </button>
                        </div>
                    </div>
                </div>
                <div class="comments-section">
                    <h2 class="comments-header">Comments</h2>

                    <!-- Rating Input Section -->
                    <div class="write-review-section" style="margin-bottom: 20px; padding: 15px; background-color: #2a2a2a; border-radius: 10px;">
                         <h3 style="color: #00ffff; font-size: 1.2em; margin-bottom: 10px;">Leave a Review</h3>
                         <div class="rating-input" style="margin-bottom: 15px;">
                             <span style="color: #ccc; margin-right: 10px;">Your rating:</span>
                              <div class="stars" id="user-rating-stars" data-movie-id="<?php echo $movieId; ?>">
                                 <i class="far fa-star" data-rating="1"></i>
                                 <i class="far fa-star" data-rating="2"></i>
                                 <i class="far fa-star" data-rating="3"></i>
                                 <i class="far fa-star" data-rating="4"></i>
                                 <i class="far fa-star" data-rating="5"></i>
                             </div>
                         </div>
                         <!-- Comment Form -->
                         <form id="comment-form" action="submit_review.php" method="POST"> <!-- Action points to AJAX endpoint -->
                             <input type="hidden" name="movie_id" value="<?php echo $movieId; ?>">
                             <input type="hidden" name="rating" id="submitted-rating" value="0"> <!-- Hidden input for rating value -->
                             <textarea name="comment" class="comment-input" placeholder="Write a comment..." required></textarea>
                              <button type="submit" class="action-button" style="background-color: #00ffff; color: #1a1a1a;">Submit Review</button>
                         </form>
                    </div>


                    <div class="comment-list">
                        <?php if (!empty($reviews)): ?>
                            <?php foreach ($reviews as $review): ?>
                            <div class="comment">
                                <div class="comment-header">
                                     <!-- Optional: User profile image next to username -->
                                     <img src="<?php echo htmlspecialchars($review['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($review['username'] ?? 'User') . '&background=random'); ?>" alt="User Avatar" style="width: 30px; height: 30px; border-radius: 50%; margin-right: 10px; vertical-align: middle;">
                                    <strong><?php echo htmlspecialchars($review['username']); ?></strong>
                                    <div class="comment-actions">
                                         <!-- Display rating for this specific review -->
                                          <span style="color: #ffd700; margin-right: 10px;">
                                            <?php
                                             $reviewRating = floatval($review['rating']);
                                             for ($i = 1; $i <= 5; $i++) {
                                                 if ($i <= $reviewRating) {
                                                     echo '<i class="fas fa-star" style="font-size: 0.9em;"></i>';
                                                 } else if ($i - 0.5 <= $reviewRating) {
                                                     echo '<i class="fas fa-star-half-alt" style="font-size: 0.9em;"></i>';
                                                 } else {
                                                     echo '<i class="far fa-star" style="font-size: 0.9em;"></i>';
                                                 }
                                             }
                                             echo " ({$reviewRating})";
                                            ?>
                                         </span>
                                        <!-- Placeholder actions -->
                                        <!-- <i class="fas fa-thumbs-up"></i> -->
                                        <!-- <i class="fas fa-thumbs-down"></i> -->
                                        <!-- <i class="fas fa-reply"></i> -->
                                    </div>
                                </div>
                                <p><?php echo nl2br(htmlspecialchars($review['comment'])); ?></p> <!-- nl2br to preserve newlines -->
                                 <div class="comment-date" style="font-size: 0.8em; color: #666; margin-top: 5px;">
                                     <?php echo date('Y-m-d H:i', strtotime($review['created_at'])); ?>
                                 </div>
                            </div>
                            <?php endforeach; ?>
                        <?php else: ?>
                            <div class="empty-state" style="padding: 20px;">
                                <i class="fas fa-comment"></i>
                                <p>No reviews yet</p>
                                <p class="subtitle">Be the first to leave a review!</p>
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
                <!-- Iframe will be populated by JS -->
                 <iframe id="trailer-iframe" src="" frameborder="0" allowfullscreen></iframe>
                 <!-- Or <video> tag for local files, needs JS logic -->
                 <!-- <video id="trailer-video" controls style="width: 100%; height: 100%; display: none;"></video> -->
            </div>
        </div>
    </div>

    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
     <script>
         document.addEventListener('DOMContentLoaded', function() {
             // The movie details data is already loaded by PHP and available in the HTML.
             // The JS in main.js for trailer modal and favorite button should work if they
             // target elements with correct classes/IDs.

             // --- User Rating Input on Details Page ---
             const userRatingStarsContainer = document.getElementById('user-rating-stars');
             const submittedRatingInput = document.getElementById('submitted-rating');
             const commentForm = document.getElementById('comment-form');

             let currentUserRating = 0; // To store the rating selected by the user before submitting

             if (userRatingStarsContainer && submittedRatingInput && commentForm) {
                 const stars = userRatingStarsContainer.querySelectorAll('i');

                 stars.forEach(star => {
                     star.addEventListener('mouseover', function() {
                         const rating = this.getAttribute('data-rating');
                         highlightRatingStars(stars, rating, 'fas');
                     });

                     star.addEventListener('mouseout', function() {
                          if (currentUserRating > 0) {
                               highlightRatingStars(stars, currentUserRating, 'fas'); // Revert to selected rating
                           } else {
                               highlightRatingStars(stars, 0, 'far'); // Clear stars if none selected
                           }
                     });

                     star.addEventListener('click', function() {
                         const rating = this.getAttribute('data-rating');
                         currentUserRating = rating; // Store the selected rating
                         submittedRatingInput.value = rating; // Update hidden input value
                         highlightRatingStars(stars, currentUserRating, 'fas'); // Highlight based on click
                         console.log("User selected rating:", currentUserRating);
                     });
                 });

                 // Helper function to highlight stars
                  function highlightRatingStars(starElements, rating, className) {
                       starElements.forEach((star, index) => {
                           star.classList.remove('fas', 'far', 'fa-star-half-alt'); // Clear existing
                           if (index < rating) {
                               star.classList.add('fas', 'fa-star');
                           } else {
                               star.classList.add('far', 'fa-star');
                           }
                       });
                  }

                 // --- Handle Comment Form Submission (AJAX) ---
                 commentForm.addEventListener('submit', function(event) {
                     event.preventDefault(); // Prevent default form submission

                     const movieId = this.querySelector('input[name="movie_id"]').value;
                     const rating = this.querySelector('input[name="rating"]').value;
                     const comment = this.querySelector('textarea[name="comment"]').value.trim();

                     if (comment === "") {
                         alert("Comment cannot be empty.");
                         return;
                     }
                     // Optional: Require a rating before submitting
                     if (rating === "0") {
                         alert("Please select a rating.");
                         return;
                     }


                     // TODO: Send data via AJAX to review/submit_review.php
                     const formData = new FormData();
                     formData.append('movie_id', movieId);
                     formData.append('rating', rating);
                     formData.append('comment', comment);

                     fetch('submit_review.php', { // Adjust path to AJAX endpoint
                         method: 'POST',
                         body: formData
                     })
                     .then(response => response.json())
                     .then(data => {
                         if (data.success) {
                             alert('Review submitted successfully!');
                             // Optional: Dynamically add the new comment to the list
                             // Requires the endpoint to return the comment data (username, text, date)
                             // Or simply reload the page to see the new comment
                             window.location.reload(); // Simple reload to show new comment

                         } else {
                             console.error('Review submission failed:', data.message);
                             alert('Failed to submit review: ' + data.message);
                         }
                     })
                     .catch(error => {
                         console.error('AJAX Error submitting review:', error);
                         alert('An error occurred while submitting your review.');
                     });
                 });

             } // End if userRatingStarsContainer exists

             // --- Favorite Button on Details Page ---
             // The event listener for `.favorite-btn` is already in main.js,
             // targeting buttons with `data-movie-id`. Ensure the button in
             // this file has that attribute. The button text/icon update logic
             // for this specific button needs to be handled as well.
             const detailsPageFavoriteBtn = document.querySelector('.movie-details-page .favorite-btn');
              if(detailsPageFavoriteBtn) {
                   // The event listener in main.js will handle the click and AJAX.
                   // Add specific logic here if the *text* needs to change ("Add" vs "Remove")
                   // based on the AJAX response.
                    detailsPageFavoriteBtn.addEventListener('click', function() {
                        // The fetch in main.js completes. It should update the button's class.
                        // Check the button's *new* state after the potential update
                        const isNowFavorited = this.classList.contains('fas'); // Assuming fas is 'added'
                        this.querySelector('span').textContent = isNowFavorited ? 'Remove from Favorites' : 'Add to Favorites';
                        // The icon (<i> tag) update is already handled in main.js by changing the fas/far class
                    });
              }


         }); // End DOMContentLoaded
     </script>
</body>
</html>
```

**18. `acc_page/update_profile.php` (Baru - AJAX Endpoint)**

Buat file `acc_page/update_profile.php` untuk menangani permintaan AJAX dari halaman profil.

```php
<?php
// acc_page/update_profile.php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php';

// Set header for JSON response
header('Content-Type: application/json');

// Check if user is logged in
if (!is_logged_in()) {
    echo json_encode(['success' => false, 'message' => 'User not logged in.']);
    exit;
}

$userId = get_user_id();
$field = $_POST['field'] ?? null;
$value = $_POST['value'] ?? null;

// Validate input
if (!$field || $value === null) {
    echo json_encode(['success' => false, 'message' => 'Invalid request.']);
    exit;
}

// Map client-side IDs to database field names
$allowedFields = [
    'displayName' => 'full_name',
    'username' => 'username',
    'bio' => 'bio'
];

if (!isset($allowedFields[$field])) {
    echo json_encode(['success' => false, 'message' => 'Invalid field specified.']);
    exit;
}

$dbField = $allowedFields[$field];

// Perform the update using the database function
try {
    $success = updateUserField($userId, $dbField, $value);

    if ($success) {
         // If updating username and value was changed, re-fetch user data to ensure username is valid/unique
         // The updateUserField function already checks for username uniqueness for *other* users.
         // If it returns true, the update was successful for this user.
         // We can return the potentially cleaned value from the database if updateUserField supported it,
         // but for simplicity, just return the submitted value on success.
         echo json_encode(['success' => true, 'message' => 'Profile updated successfully.', 'value' => $value]);
    } else {
         // Check specifically for username uniqueness failure if that was the field
         if ($dbField === 'username') {
              // Need a way for updateUserField to signal *why* it failed (e.g., username exists)
              // For now, a generic "failed" message is okay, or modify updateUserField to return a specific code/message
               // A simpler check here: if update failed for username, assume it's because it exists
               $currentUserData = getUserById($userId); // Get current data
               if ($currentUserData['username'] !== $value) { // If the user *intended* to change it
                   $stmtCheck = $pdo->prepare("SELECT COUNT(*) FROM users WHERE username = ?");
                   $stmtCheck->execute([$value]);
                   if ($stmtCheck->fetchColumn() > 0) {
                       echo json_encode(['success' => false, 'message' => 'Username already taken.']);
                       exit;
                   }
               }
               // If not username or uniqueness issue not detected, generic fail
               echo json_encode(['success' => false, 'message' => 'Failed to update profile.']);

         } else {
              echo json_encode(['success' => false, 'message' => 'Failed to update profile.']);
         }

    }

} catch (PDOException $e) {
    error_log("Database error updating user profile ({$dbField}): " . $e->getMessage());
    echo json_encode(['success' => false, 'message' => 'An internal error occurred.']);
}

?>
```

**19. `favorite/toggle_favorite.php` (Baru - AJAX Endpoint)**

Buat file `favorite/toggle_favorite.php` untuk menangani permintaan AJAX favorite.

```php
<?php
// favorite/toggle_favorite.php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php';

header('Content-Type: application/json');

if (!is_logged_in()) {
    echo json_encode(['success' => false, 'message' => 'User not logged in.']);
    exit;
}

$userId = get_user_id();
$movieId = $_POST['movie_id'] ?? null;
$action = $_POST['action'] ?? null; // 'add' or 'remove'

if (!$movieId || !in_array($action, ['add', 'remove'])) {
    echo json_encode(['success' => false, 'message' => 'Invalid request.']);
    exit;
}

$movieId = (int)$movieId; // Ensure movie ID is integer

try {
    if ($action === 'add') {
        $success = addToFavorites($movieId, $userId);
        if ($success) {
             echo json_encode(['success' => true, 'action' => 'added', 'message' => 'Added to favorites.']);
        } else {
             echo json_encode(['success' => false, 'message' => 'Failed to add to favorites.']);
        }
    } else { // action is 'remove'
        $success = removeFromFavorites($movieId, $userId);
         // Removing a non-existent favorite is often considered success
         echo json_encode(['success' => true, 'action' => 'removed', 'message' => 'Removed from favorites.']);
         // Or check $success for true/false if you need stricter feedback
    }

} catch (PDOException $e) {
    error_log("Database error toggling favorite ({$action}): " . $e->getMessage());
    echo json_encode(['success' => false, 'message' => 'An internal error occurred.']);
}
?>
```

**20. `review/submit_review.php` (Baru - AJAX Endpoint)**

Buat file `review/submit_review.php` untuk menangani permintaan AJAX submit review.

```php
<?php
// review/submit_review.php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth_helper.php';

header('Content-Type: application/json');

if (!is_logged_in()) {
    echo json_encode(['success' => false, 'message' => 'User not logged in.']);
    exit;
}

$userId = get_user_id();
$movieId = $_POST['movie_id'] ?? null;
$rating = $_POST['rating'] ?? null;
$comment = trim($_POST['comment'] ?? '');

// Validate input
if (!$movieId || $rating === null || empty($comment)) {
    echo json_encode(['success' => false, 'message' => 'Movie ID, rating, and comment are required.']);
    exit;
}

$movieId = (int)$movieId;
$rating = floatval($rating);

// Validate rating value range (e.g., 1.0 to 5.0)
if ($rating < 1.0 || $rating > 5.0) {
     echo json_encode(['success' => false, 'message' => 'Invalid rating value. Rating must be between 1.0 and 5.0']);
     exit;
}

try {
    // Check if the user has already reviewed this movie (optional, based on desired logic)
    // If they can only submit one review, you'd check here.
    // For simplicity, current createReview allows multiple reviews.

    $success = createReview($movieId, $userId, $rating, $comment);

    if ($success) {
        // Optionally, fetch the newly created review or the updated average rating to return
        // For simplicity, just return success status. Client can reload or update UI.
        echo json_encode(['success' => true, 'message' => 'Review submitted successfully.']);
    } else {
        echo json_encode(['success' => false, 'message' => 'Failed to submit review.']);
    }

} catch (PDOException $e) {
    error_log("Database error submitting review: " . $e->getMessage());
    echo json_encode(['success' => false, 'message' => 'An internal error occurred.']);
}
?>
```

**21. `manage/upload_handler.php` (Opsional Handler)**

Anda bisa memproses form upload langsung di `manage/upload.php` seperti yang sudah dilakukan di modifikasi step 14. Jadi file handler terpisah ini tidak wajib jika semua logika ada di file form itu sendiri. Logika upload file di modifikasi step 14 sudah cukup untuk contoh ini.

---

**Langkah Selanjutnya:**

1.  Buat struktur folder seperti yang dijelaskan.
2.  Buat/Copy/Modifikasi semua file PHP dan JS sesuai kode di atas.
3.  Gabungkan semua CSS ke `css/style.css` dan sesuaikan path gambar latar serta warna.
4.  Pastikan folder `uploads/posters/`, `uploads/profiles/`, `uploads/trailers/` ada dan memiliki izin tulis untuk server web Anda.
5.  Pastikan database `ratingtales` dan semua tabelnya sudah dibuat sesuai skema SQL yang Anda berikan.
6.  Ubah `YOUR_GOOGLE_CLIENT_ID` di `form-login.php` jika Anda ingin mengaktifkan Google Sign-In (memerlukan konfigurasi tambahan di Google Cloud Console).
7.  Sesuaikan path root `/ratingtales/` dalam `require_login()` di `auth_helper.php` dan link-link sidebar jika folder instalasi Anda berbeda dari `ratingtales`.
8.  Akses aplikasi melalui root folder Anda (misalnya, `http://localhost/ratingtales/`). Anda seharusnya diarahkan ke halaman login.

Dengan mengikuti langkah-langkah ini, Anda akan memiliki aplikasi web yang terintegrasi dengan database, menerapkan fungsionalitas inti yang Anda minta, dan menjaga tampilan desain asli, semua dalam satu file CSS.
