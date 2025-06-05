Baik, mari kita perbaiki cacat-cacat yang Anda sebutkan dengan menyesuaikan kode program yang sudah ada.

Berdasarkan poin-poin yang Anda sampaikan dan kode terakhir yang kita bahas, berikut adalah area yang akan kita fokuskan:

1.  **Login Username/Email:** Memastikan logika PHP di `autentikasi/form-login.php` dan fungsi `getUserByUsernameOrEmail` di `config/database.php` bekerja dengan benar untuk otentikasi manual.
2.  **Link Kelola Film:** Menambahkan kembali link "Manage" di sidebar pada halaman yang menampilkan sidebar (`beranda/index.php`, `favorite/index.php`, `review/index.php`, `acc_page/index.php`).
3.  **Trailer:** Memperbaiki fungsi JavaScript di `js/main.js` dan struktur HTML di `review/movie-details.php` agar dapat memutar trailer baik dari URL YouTube (`trailer_link`) maupun dari file yang diunggah (`trailer_file`).
4.  **Gambar Profil:** Menyatakan kembali bahwa fitur *mengganti* gambar profil melalui klik icon belum diimplementasikan. Kode yang ada hanya menampilkan gambar profil dan mengedit teks. Kita akan memastikan tampilan gambar profil dari DB sudah benar.
5.  **Rekomendasi/For You:** Menambahkan penanganan *empty state* (pesan "tidak ada film") di halaman beranda jika tabel `movies` kosong.

Kita akan berasumsi struktur folder Anda sesuai dengan rekomendasi terakhir (`config/database.php`, `includes/auth_helper.php`, `css/style.css`, `js/main.js`, dll.) dan Anda telah menghapus kode/file terkait CAPTCHA dan Google Sign-In.

Berikut adalah kode lengkap untuk file-file yang perlu diubah:

**1. `config/database.php` (Pastikan Sesuai Versi Tanpa CAPTCHA/Google)**

Kode ini seharusnya sudah sesuai dengan versi yang kita sepakati setelah menghapus CAPTCHA/Google. Pastikan fungsi user (`createUser`, `getUserById`, `getUserByUsernameOrEmail`, `updateUserField`) sudah sesuai dengan kolom di tabel `users` Anda, dan fungsi `generateRandomString` sudah dihapus.

*(Kode `config/database.php` tetap sama seperti jawaban sebelumnya yang sudah menghilangkan CAPTCHA/Google. Saya tidak menempelkannya lagi di sini untuk menghindari redudansi masif).*

**2. `includes/auth_helper.php` (Pastikan Ada dan Sesuai)**

File ini berisi fungsi `is_logged_in`, `require_login`, `get_user_id`, `get_logged_in_user_data`. Pastikan file ini ada di folder `includes/` Anda.

*(Kode `includes/auth_helper.php` tetap sama seperti jawaban sebelumnya. Saya tidak menempelkannya lagi di sini).*

**3. `css/style.css` (Hapus Style CAPTCHA/Google)**

Pastikan Anda telah menghapus semua blok kode CSS yang secara spesifik menargetkan elemen CAPTCHA dan Google Sign-In seperti yang dijelaskan sebelumnya.

*(Kode `css/style.css` tetap sama seperti jawaban sebelumnya setelah menghapus style CAPTCHA/Google. Saya tidak menempelkannya lagi di sini).*

**4. `js/animation.js` (Untuk Animasi Form Auth)**

File ini berisi JavaScript untuk animasi form login/register.

*(Kode `js/animation.js` tetap sama. Saya tidak menempelkannya lagi di sini).*

**5. `js/main.js` (Modifikasi Penanganan Trailer)**

Kita akan memodifikasi fungsi `playTrailer` di sini agar dapat membedakan antara URL YouTube dan path file lokal, dan menampilkan modal trailer dengan elemen yang sesuai.

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
    // Note: This search is client-side filtering based on current HTML content.
    const searchInput = document.querySelector('.search-bar input');
    const searchButton = document.querySelector('.search-bar button'); // Assuming a search button exists

    if (searchInput) {
        // Function to perform search filtering on movie/item cards
        function performSearch() {
            const searchTerm = searchInput.value.toLowerCase();
            // Target grid containers on different pages
            const grid = document.querySelector('.review-grid, .movies-grid, .movie-grid');
            const cards = document.querySelectorAll('.movie-card'); // Adjust selector if needed

            let visibleCardCount = 0;

            cards.forEach(card => {
                const titleElement = card.querySelector('h3');
                // Find info elements - might be .movie-info or others depending on card structure
                const infoElements = card.querySelectorAll('.movie-info, .movie-details p:not(.movie-info)');

                const title = titleElement ? titleElement.textContent.toLowerCase() : '';
                let info = '';
                infoElements.forEach(el => { info += el.textContent.toLowerCase() + ' '; });
                info = info.trim();


                if (title.includes(searchTerm) || info.includes(searchTerm)) {
                    card.style.display = '';
                    visibleCardCount++;
                } else {
                    card.style.display = 'none';
                }
            });

            // Handle empty state messages based on search results
            const initialEmptyState = document.querySelector('.empty-state:not(.search-empty-state)'); // Get initial empty state
            const searchEmptyState = document.querySelector('.search-empty-state'); // Get search empty state

            if (visibleCardCount === 0 && searchTerm !== '') {
                 // Hide initial empty state if searching
                 if(initialEmptyState) initialEmptyState.style.display = 'none';

                 if (!searchEmptyState) {
                     // Create and append search empty state if it doesn't exist
                     if (grid) {
                         const emptyState = document.createElement('div');
                         emptyState.className = 'empty-state search-empty-state full-width'; // Add full-width class if needed by CSS
                         let message = 'No results found';
                         let subtitle = 'Try a different search term';
                         // Customize message based on page (can add body class to differentiate pages)
                         if (document.body.classList.contains('favorites-page')) {
                             message = 'Film favorit tidak ditemukan.';
                             subtitle = 'Coba kata kunci pencarian lain.';
                         } else if (document.body.classList.contains('manage-page')) {
                             message = 'Film tidak ditemukan.';
                             subtitle = 'Coba kata kunci pencarian lain.';
                         } else if (document.body.classList.contains('review-page')) {
                              message = 'Film tidak ditemukan.';
                              subtitle = 'Coba kata kunci pencarian lain.';
                         } else if (document.body.classList.contains('home-page')) {
                              // Search might not be prominent on home, but good to handle
                               message = 'Film tidak ditemukan.';
                               subtitle = 'Coba kata kunci pencarian lain.';
                         }


                         emptyState.innerHTML = `
                             <i class="fas fa-search"></i>
                             <p>${htmlspecialcharsJS(message)}</p>
                             <p class="subtitle">${htmlspecialcharsJS(subtitle)}</p>
                         `;
                         grid.appendChild(emptyState);
                     }
                 } else {
                     // Update text and show search empty state if it exists
                     searchEmptyState.querySelector('p:first-of-type').innerText = `Film tidak ditemukan matching "${htmlspecialcharsJS(searchTerm)}"`; // Update text
                     searchEmptyState.style.display = 'flex';
                 }
            } else {
                // Remove search empty state if cards are visible or search is cleared
                if (searchEmptyState) {
                    searchEmptyState.remove();
                }
                 // Show initial empty state if no movies were loaded AND search is cleared
                 // This requires checking the *initial* number of cards loaded by PHP
                 // For simplicity, we check if ANY cards are currently displayed.
                if (cards.length === 0 && searchTerm === '' && initialEmptyState) {
                     initialEmptyState.style.display = 'flex'; // Show initial empty state if no cards were initially loaded
                } else if (initialEmptyState) {
                     initialEmptyState.style.display = 'none'; // Hide initial empty state if there are movies or search results
                }
            }
        }

        // Add event listeners for search
        searchInput.addEventListener('input', performSearch);
        if (searchButton) {
            searchButton.addEventListener('click', function(e) {
                e.preventDefault(); // Prevent form submission if button is in a form
                performSearch(); // Trigger search on button click
            });
        }

        // Helper function for HTML escaping strings in JS before putting into innerHTML
        function htmlspecialcharsJS(str) {
             if (typeof str !== 'string') return str;
             const map = {
                 '&': '&amp;',
                 '<': '&lt;',
                 '>': '&gt;',
                 '"': '&quot;',
                 "'": '&#039;'
             };
             return str.replace(/[&<>"']/g, function(m) { return map[m]; });
         }

    } // End if searchInput exists


    // --- Review Page Popup and Rating ---
    const movieDetailsPopup = document.getElementById('movie-details-popup');
    let currentMovieData = {}; // To store data for the popup/details page

    // Function to open the popup with movie data
    window.showMovieDetailsPopup = function(movieId, title, year, genre, description, rating, trailerUrl) {
         currentMovieData = {
             id: movieId,
             title: title,
             year: year,
             genre: genre,
             description: description, // Assuming description is already HTML escaped if needed
             rating: rating,
             trailerUrl: trailerUrl // Pass trailer URL here
         };

        document.getElementById('popup-title').textContent = currentMovieData.title;
        document.getElementById('popup-year-genre').textContent = currentMovieData.year + ' | ' + currentMovieData.genre;
        document.getElementById('popup-description').textContent = currentMovieData.description;
        const avgRatingSpan = document.querySelector('#movie-details-popup .popup-rating span');
        if (avgRatingSpan) {
            avgRatingSpan.textContent = `${currentMovieData.rating}/5`; // Display average rating
        }


        const popupRatingStarsContainer = document.querySelector('#movie-details-popup .rating-stars');
        if (popupRatingStarsContainer) {
             // Clear existing stars and re-add for click interaction
             popupRatingStarsContainer.innerHTML = '';
             for (let i = 1; i <= 5; i++) {
                  const star = document.createElement('i');
                  star.className = 'far fa-star'; // Start with empty stars for user to click
                  star.setAttribute('data-rating', i);
                  popupRatingStarsContainer.appendChild(star);
             }

             // Add event listeners for rating input within the popup
             popupRatingStarsContainer.querySelectorAll('i').forEach(star => {
                 star.addEventListener('click', handlePopupRatingClick); // Use specific handler for popup
                 star.addEventListener('mouseover', handlePopupRatingHover);
                 star.addEventListener('mouseout', handlePopupRatingOut);
             });

             // Reset user selected rating state when opening popup
             currentUserSelectedRating = 0;
             highlightPopupStars(popupRatingStarsContainer.querySelectorAll('i'), currentUserSelectedRating, 'far'); // Ensure starts are empty initially


        }

        // Update the favorite button in the popup based on the original card's state
        const cardFavoriteBtn = document.querySelector(`.movie-card[data-movie-id="${currentMovieData.id}"] .favorite-btn`);
        const popupFavoriteBtn = document.querySelector('.popup-content .add-favorite-popup-btn');
        if(popupFavoriteBtn && cardFavoriteBtn) {
             if (cardFavoriteBtn.classList.contains('fas')) {
                 popupFavoriteBtn.innerHTML = '<i class="fas fa-heart"></i> Remove from Favorites';
             } else {
                 popupFavoriteBtn.innerHTML = '<i class="far fa-heart"></i> Add to Favorites';
             }
             // The event listener for this button is handled by the general .favorite-btn listener below
             // which uses event delegation or targets all matching elements.
             // Ensure this button also has data-movie-id or the listener finds it correctly.
             popupFavoriteBtn.setAttribute('data-movie-id', currentMovieData.id); // Add movie ID to button
        }


        // Update Watch Trailer button in popup
        const popupWatchTrailerBtn = document.querySelector('.popup-content .watch-trailer');
        if(popupWatchTrailerBtn) {
             // Remove old onclick and add event listener
             // Use a dedicated event listener to call the main window.playTrailer function
             popupWatchTrailerBtn.onclick = null; // Remove inline click handler
             // Ensure listener is added only once - check if it's already attached or remove it first
             // For simplicity now, we just add the listener. A better approach is to manage listeners.
             popupWatchTrailerBtn.addEventListener('click', function() {
                  // Call the main playTrailer function with the trailer URL
                  if (currentMovieData.trailerUrl) {
                       window.playTrailer(currentMovieData.trailerUrl); // Call the main function
                  } else {
                       alert("Trailer tidak tersedia untuk film ini.");
                  }
             });
        }


        if (movieDetailsPopup) {
            movieDetailsPopup.classList.add('active');
        }

        // Close popup when clicking outside
        if (movieDetailsPopup) {
             // Remove previous click listener to prevent duplicates
             if (movieDetailsPopup._clickHandler) { // Check if a handler was previously stored
                 movieDetailsPopup.removeEventListener('click', movieDetailsPopup._clickHandler);
             }
             const newClickHandler = function(e) {
                 // Check if click target is the modal background itself, not the content
                 if (e.target === movieDetailsPopup) {
                     hideMovieDetailsPopup();
                 }
             };
             movieDetailsPopup.addEventListener('click', newClickHandler);
             movieDetailsPopup._clickHandler = newClickHandler; // Store for removal later
        }
    }

    // Function to hide the popup
    window.hideMovieDetailsPopup = function() {
        if (movieDetailsPopup) {
            movieDetailsPopup.classList.remove('active');
             // Reset trailer modal state in case it was opened from here
             window.closeTrailer();
        }
    }


    // Handle Rating Interaction in Popup (Client-side visual feedback)
    let currentUserSelectedRating = 0; // Store user's potential rating

    function handlePopupRatingHover() {
        const rating = this.getAttribute('data-rating');
        const stars = this.parentElement.querySelectorAll('i');
        highlightPopupStars(stars, rating, 'fas');
    }

     function handlePopupRatingOut() {
         // Reset to user's selected rating, or clear if none selected
         const stars = this.parentElement.querySelectorAll('i');
          if (currentUserSelectedRating > 0) {
               highlightPopupStars(stars, currentUserSelectedRating, 'fas');
          } else {
               highlightPopupStars(stars, 0, 'far'); // Clear all stars
          }
     }

    function handlePopupRatingClick() {
        const clickedRating = this.getAttribute('data-rating');
        currentUserSelectedRating = clickedRating; // Store the selected rating
        const stars = this.parentElement.querySelectorAll('i');
        highlightPopupStars(stars, currentUserSelectedRating, 'fas'); // Highlight based on click
        console.log(`User selected ${currentUserSelectedRating} stars for movie ID: ${currentMovieData.id}`);

        // Optional: Automatically trigger comment form rating input update
         const submittedRatingInput = document.getElementById('submitted-rating'); // This is on the movie-details page, not popup
         // If you want to submit rating from popup, you need a form/AJAX handler here

        // TODO: If you want to submit the rating ONLY from the popup, add AJAX call here.
        // For now, the rating input is primarily designed for the movie-details page comment form.
        alert(`You selected ${currentUserSelectedRating} stars!`); // Placeholder feedback
    }

     function highlightPopupStars(starElements, rating, className) {
          starElements.forEach((star, index) => {
               star.classList.remove('fas', 'far', 'fa-star-half-alt'); // Clear existing
               if (index < rating) {
                   star.classList.add('fas', 'fa-star');
               } else {
                   star.classList.add('far', 'fa-star');
               }
           });
     }


    // "Read More" button in Review Popup - Redirects to movie-details page
    window.openMovieDetails = function() {
        if (currentMovieData && currentMovieData.id) {
            window.location.href = `movie-details.php?id=${currentMovieData.id}`; // Adjust path if needed
        } else {
            console.error("Cannot open movie details: movie ID is missing.");
             alert("Tidak dapat membuka detail film."); // User feedback
        }
    }

    // --- Movie Details Page Trailer Modal ---
    // This modal and the functions should be reusable by both the popup and the full details page
    const trailerModal = document.getElementById('trailer-modal');
    const trailerIframe = document.getElementById('trailer-iframe');
    const trailerVideo = document.getElementById('trailer-video'); // Assuming a video tag exists
    const closeTrailerButton = document.querySelector('.close-trailer');

    if (trailerModal && (trailerIframe || trailerVideo)) {
        window.playTrailer = function(trailerUrl) {
            if (!trailerUrl) {
                 alert("Trailer tidak tersedia untuk film ini.");
                 return;
            }

            // Hide both elements initially
            if (trailerIframe) trailerIframe.style.display = 'none';
            if (trailerVideo) trailerVideo.style.display = 'none';

            // Check if it's likely a YouTube URL
            if (trailerUrl.includes('youtube.com/watch') || trailerUrl.includes('youtu.be/')) {
                if (trailerIframe) {
                     let embedUrl = trailerUrl;
                     if (trailerUrl.includes('watch?v=')) {
                          const videoId = trailerUrl.split('v=')[1].split('&')[0];
                          embedUrl = `https://www.youtube.com/embed/${videoId}`;
                     } else if (trailerUrl.includes('youtu.be/')) {
                          const videoId = trailerUrl.split('/').pop();
                          embedUrl = `https://www.youtube.com/embed/${videoId}`;
                     }
                     // Add parameters for autoplay and related videos off
                     trailerIframe.src = `${embedUrl}?autoplay=1&rel=0`;
                     trailerIframe.style.display = 'block'; // Show the iframe
                } else {
                    console.error("Trailer is a YouTube URL, but iframe element (#trailer-iframe) is missing.");
                    alert("Tidak dapat memutar trailer (elemen player tidak ditemukan).");
                    return;
                }

            } else {
                // Assuming it's a local file path or other embed type
                // For local files, use the <video> tag
                if (trailerVideo) {
                     trailerVideo.src = trailerUrl;
                     trailerVideo.style.display = 'block'; // Show the video tag
                     trailerVideo.load(); // Load the video source
                     trailerVideo.play().catch(error => {
                          console.error("Video autoplay failed:", error);
                          // Autoplay might be blocked, user needs to click play
                     });
                } else {
                     console.error("Trailer is a local file path, but video element (#trailer-video) is missing.");
                      // Fallback: Offer to open the file directly if video tag is missing
                     if (confirm("Trailer adalah file video. Buka di tab baru?")) {
                          window.open(trailerUrl, '_blank');
                     }
                     return;
                }
            }

            // Show the modal
            trailerModal.classList.add('active');
        }

        window.closeTrailer = function() {
            // Stop the video/iframe by clearing the source
            if (trailerIframe) trailerIframe.src = '';
            if (trailerVideo) {
                 trailerVideo.pause();
                 trailerVideo.src = ''; // Clear video source
             }
            // Hide the modal
            trailerModal.classList.remove('active');
        }

        // Close modal when clicking outside
         if (trailerModal) {
             // Remove previous click listener to prevent duplicates
             if (trailerModal._clickHandler) {
                 trailerModal.removeEventListener('click', trailerModal._clickHandler);
             }
             const newClickHandler = function(e) {
                 if (e.target === trailerModal) {
                     closeTrailer();
                 }
             };
             trailerModal.addEventListener('click', newClickHandler);
             trailerModal._clickHandler = newClickHandler; // Store for removal later
        }

        if (closeTrailerButton) {
             closeTrailerButton.addEventListener('click', closeTrailer);
        }
    }


    // --- Profile Page Edit Functionality ---
    // toggleEdit function is already in js/main.js
    // It sends AJAX calls to acc_page/update_profile.php (needs implementation)
    // Update_profile_picture is a separate feature, not included here.


    // --- Favorite/Review Action Buttons ---
    // These are action buttons often found on movie cards (favorite icon, etc.)
    // We use event delegation or target all matching buttons.
    // Listener for Favorite toggle (assuming a heart icon button with class 'favorite-btn')
    document.querySelectorAll('.action-btn.favorite-btn').forEach(button => {
        // Remove old onclick if any and add robust event listener
        button.onclick = null; // Remove inline onclick
        button.addEventListener('click', function(e) {
            e.stopPropagation(); // Prevent card click event from firing
            e.preventDefault(); // Prevent default button behavior

            const movieCard = this.closest('.movie-card');
            const movieId = movieCard ? movieCard.getAttribute('data-movie-id') : null;

             if (!movieId) {
                  console.error("Movie ID not found on card for favorite toggle.");
                  alert("Tidak dapat menambahkan/menghapus favorit (ID film tidak ditemukan).");
                  return;
             }

             // Determine current state (is it currently favorited?) - by checking icon class
             const iconElement = this.querySelector('i');
             const isFavorited = iconElement ? iconElement.classList.contains('fas') : false;

             // Determine action based on current state
             const action = isFavorited ? 'remove' : 'add';

             // Send AJAX request to favorite/toggle_favorite.php
             fetch('../favorite/toggle_favorite.php', { // Adjust path relative to current page
                 method: 'POST',
                 headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                 body: `movie_id=${encodeURIComponent(movieId)}&action=${encodeURIComponent(action)}`
             })
             .then(response => response.json())
             .then(data => {
                 if (data.success) {
                     console.log("Favorite toggle successful:", data.message);
                     // Update icon visually
                     if (iconElement) {
                          if (data.action === 'added') {
                              iconElement.classList.remove('far');
                              iconElement.classList.add('fas');
                              // alert("Added to favorites!"); // User feedback - optional
                          } else { // Removed
                              iconElement.classList.remove('fas');
                              iconElement.classList.add('far');
                               // alert("Removed from favorites!"); // User feedback - optional

                               // On the favorites page, remove the card from the DOM
                               if (document.body.classList.contains('favorites-page') && movieCard) {
                                   movieCard.remove();
                                    // Check if the grid is now empty and show empty state
                                   const grid = document.querySelector('.review-grid'); // Assuming favorites uses review-grid
                                   if (grid) {
                                        const remainingCards = grid.querySelectorAll('.movie-card');
                                        const initialEmptyState = document.querySelector('.empty-state:not(.search-empty-state)');
                                        const searchEmptyState = document.querySelector('.search-empty-state');

                                        if (remainingCards.length === 0) {
                                             // If search is active, show search empty state
                                             const searchInput = document.querySelector('.search-bar input');
                                             if (searchInput && searchInput.value.trim() !== '') {
                                                 // Trigger search to update empty state message
                                                  const event = new Event('input');
                                                  searchInput.dispatchEvent(event);
                                             } else {
                                                  // If no search term, show the default empty state
                                                  if(initialEmptyState) initialEmptyState.style.display = 'flex';
                                                  if(searchEmptyState) searchEmptyState.style.display = 'none';
                                             }
                                        } else {
                                            // If there are still cards, ensure empty states are hidden
                                             if(initialEmptyState) initialEmptyState.style.display = 'none';
                                             if(searchEmptyState) searchEmptyState.style.display = 'none';
                                        }
                                   }
                               }
                           }
                       // Update popup button text/icon if the popup is currently open for this movie
                       const popupFavoriteBtn = document.querySelector('.movie-details-popup .add-favorite-popup-btn');
                       if (movieDetailsPopup && movieDetailsPopup.classList.contains('active') && currentMovieData.id == movieId && popupFavoriteBtn) {
                           if (data.action === 'added') {
                               popupFavoriteBtn.innerHTML = '<i class="fas fa-heart"></i> Remove from Favorites';
                           } else {
                               popupFavoriteBtn.innerHTML = '<i class="far fa-heart"></i> Add to Favorites';
                           }
                       }

                     }
                 } else {
                     console.error("Failed to toggle favorite:", data.message);
                     alert("Gagal memperbarui favorit: " + data.message); // User feedback
                 }
             })
             .catch(error => {
                 console.error("AJAX Error toggling favorite:", error);
                 alert("Terjadi kesalahan saat memperbarui favorit."); // Generic error
             });
        });
    });

    // --- Movie Details Page User Rating Input ---
    // This is different from the rating display in the popup
    const userRatingStarsContainerDetails = document.getElementById('user-rating-stars');
    const submittedRatingInput = document.getElementById('submitted-rating');
    const commentForm = document.getElementById('comment-form');


    if (userRatingStarsContainerDetails && submittedRatingInput) {
        const stars = userRatingStarsContainerDetails.querySelectorAll('i');
        let currentUserRating = 0; // Store user's selected rating on this page

        stars.forEach(star => {
             star.addEventListener('mouseover', function() {
                 const rating = this.getAttribute('data-rating');
                 highlightRatingStarsDetails(stars, rating);
             });

             star.addEventListener('mouseout', function() {
                 // Revert to user's selected rating or clear if none selected
                  highlightRatingStarsDetails(stars, currentUserRating);
             });

             star.addEventListener('click', function() {
                 const clickedRating = this.getAttribute('data-rating');
                 currentUserRating = clickedRating; // Store the selected rating
                 submittedRatingInput.value = clickedRating; // Update hidden input value
                 highlightRatingStarsDetails(stars, currentUserRating); // Highlight based on click
                 console.log("User selected rating:", currentUserRating);
                 // No AJAX sent on click here, it's sent with the comment form submission
             });
         });

         // Helper function to highlight stars on the details page rating input
          function highlightRatingStarsDetails(starElements, rating) {
               starElements.forEach((star, index) => {
                   star.classList.remove('fas', 'far', 'fa-star-half-alt');
                   if (index < rating) {
                       star.classList.add('fas', 'fa-star');
                   } else {
                       star.classList.add('far', 'fa-star');
                   }
               });
          }

         // --- Handle Comment Form Submission (AJAX) ---
         if (commentForm) {
              commentForm.addEventListener('submit', function(event) {
                  event.preventDefault(); // Prevent default form submission

                  const movieId = this.querySelector('input[name="movie_id"]').value;
                  const rating = this.querySelector('input[name="rating"]').value; // Get rating from hidden input
                  const comment = this.querySelector('textarea[name="comment"]').value.trim();

                  if (comment === "") {
                      alert("Komentar tidak boleh kosong.");
                      return;
                  }
                  // Optional: Require a rating before submitting
                  if (rating === "0") {
                      alert("Silakan pilih rating.");
                      return;
                  }

                  const formData = new FormData();
                  formData.append('movie_id', movieId);
                  formData.append('rating', rating);
                  formData.append('comment', comment);

                  // Send data via AJAX to review/submit_review.php
                  fetch('submit_review.php', { // Path relative to movie-details.php
                      method: 'POST',
                      body: formData
                  })
                  .then(response => response.json())
                  .then(data => {
                      if (data.success) {
                          alert('Review berhasil dikirim!');
                          // Clear the form
                          commentForm.reset();
                          currentUserSelectedRating = 0; // Reset client-side rating state
                          highlightRatingStarsDetails(stars, 0); // Clear rating stars visual
                          // Optional: Dynamically add the new comment to the list
                          // For simplicity, reload the page to show the new comment
                          window.location.reload();

                      } else {
                          console.error('Review submission failed:', data.message);
                          alert('Gagal mengirim review: ' + data.message);
                      }
                  })
                  .catch(error => {
                      console.error('AJAX Error submitting review:', error);
                      alert('Terjadi kesalahan saat mengirim review Anda.');
                  });
              });
         } // End if commentForm exists


    } // End if userRatingStarsContainerDetails exists


}); // End DOMContentLoaded
```

**6. `beranda/index.php` (Modifikasi Sidebar Link dan Empty State)**

Tambahkan link "Manage" di sidebar dan penanganan empty state untuk list film.

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
$movies = getAllMovies(); // Assumes getAllMovies orders by creation date desc

// Take the first few movies for the slider (assuming getAllMovies orders by creation date desc)
$featuredMovies = array_slice($movies, 0, min(5, count($movies))); // Get up to 5 for slider

// For trending/for you, just reuse the $movies list for now as original did
$trendingMovies = $movies; // All movies for trending
$forYouMovies = $movies; // All movies for for you (you might want different logic here later)

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Home</title>
    <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Use 6.0.0 -->
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
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
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
                             // Get year from release_date, handle potential null/invalid date
                             $releaseYear = (isset($movie['release_date']) && $movie['release_date']) ? date('Y', strtotime($movie['release_date'])) : 'N/A';
                             // Join genres array into a string, handle potential null or empty array
                             $genresString = implode(', ', $movie['genres'] ?? []);
                             if (empty($genresString)) $genresString = 'N/A';

                             // Use htmlspecialchars with ENT_QUOTES and UTF-8 for safety
                             $movieTitle = htmlspecialchars($movie['title'] ?? 'Untitled Movie', ENT_QUOTES, 'UTF-8');
                             $posterImage = htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg', ENT_QUOTES, 'UTF-8');
                         ?>
                        <div class="slide <?php echo $index === 0 ? 'active' : ''; ?>">
                            <img src="<?php echo $posterImage; ?>" alt="<?php echo $movieTitle; ?>">
                            <div class="movie-info">
                                <h1><?php echo $movieTitle; ?></h1>
                                <p><?php echo $releaseYear; ?> | <?php echo $genresString; ?></p>
                            </div>
                        </div>
                        <?php endforeach; ?>
                    <?php else: ?>
                         <!-- Empty State for Slider -->
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
                                $releaseYear = (isset($movie['release_date']) && $movie['release_date']) ? date('Y', strtotime($movie['release_date'])) : 'N/A';
                                $genresString = implode(', ', $movie['genres'] ?? []);
                                if (empty($genresString)) $genresString = 'N/A';
                                $movieTitle = htmlspecialchars($movie['title'] ?? 'Untitled Movie', ENT_QUOTES, 'UTF-8');
                                $posterImage = htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg', ENT_QUOTES, 'UTF-8');
                            ?>
                            <div class="movie-card" data-movie-id="<?php echo htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8'); ?>" onclick="window.location.href='../review/movie-details.php?id=<?php echo htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8'); ?>'"> <!-- Link to details page -->
                                <img src="<?php echo $posterImage; ?>" alt="<?php echo $movieTitle; ?>">
                                <div class="movie-details">
                                    <h3><?php echo $movieTitle; ?></h3>
                                    <p><?php echo $releaseYear; ?> | <?php echo $genresString; ?></p>
                                </div>
                            </div>
                            <?php endforeach; ?>
                         <?php else: ?>
                             <!-- Empty State for Trending -->
                             <div class="empty-state" style="min-width: 300px;">
                                 <i class="fas fa-film"></i>
                                 <p>No trending movies found</p>
                                  <p class="subtitle">Upload movies in the <a href="../manage/index.php">Manage</a> section.</p>
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
                                 $releaseYear = (isset($movie['release_date']) && $movie['release_date']) ? date('Y', strtotime($movie['release_date'])) : 'N/A';
                                 $genresString = implode(', ', $movie['genres'] ?? []);
                                 if (empty($genresString)) $genresString = 'N/A';
                                 $movieTitle = htmlspecialchars($movie['title'] ?? 'Untitled Movie', ENT_QUOTES, 'UTF-8');
                                 $posterImage = htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg', ENT_QUOTES, 'UTF-8');
                             ?>
                            <div class="movie-card" data-movie-id="<?php echo htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8'); ?>" onclick="window.location.href='../review/movie-details.php?id=<?php echo htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8'); ?>'"> <!-- Link to details page -->
                                <img src="<?php echo $posterImage; ?>" alt="<?php echo $movieTitle; ?>">
                                <div class="movie-details">
                                    <h3><?php echo $movieTitle; ?></h3>
                                    <p><?php echo $releaseYear; ?> | <?php echo $genresString; ?></p>
                                </div>
                            </div>
                            <?php endforeach; ?>
                         <?php else: ?>
                            <!-- Empty State for For You -->
                            <div class="empty-state" style="min-width: 300px;">
                                <i class="fas fa-film"></i>
                                <p>No movies for you found</p>
                                 <p class="subtitle">Upload movies in the <a href="../manage/index.php">Manage</a> section.</p>
                             </div>
                         <?php endif; ?>
                    </div>
                </div>
            </section>
        </main>
        <!-- Link to Profile Page -->
        <div class="user-profile" onclick="window.location.href='../acc_page/index.php'">
            <!-- Handle potential null profile image path -->
            <img src="<?php echo htmlspecialchars($loggedInUser['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($loggedInUser['username'] ?? 'User') . '&background=random') ; ?>" alt="User Profile" class="profile-pic">
            <span><?php echo htmlspecialchars($loggedInUser['username'] ?? 'Guest'); ?></span>
        </div>
    </div>
    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
</body>
</html>
```

**7. `favorite/index.php` (Modifikasi Sidebar Link dan Empty State)**

Tambahkan link "Manage" dan "Profile" di sidebar, perbaiki penanganan empty state untuk list favorit dan pencarian.

```php
<?php
// favorite/index.php
require_once __DIR__ . '/../config/database.php'; // Adjust path based on your config file location
require_once __DIR__ . '/../includes/auth_helper.php'; // Assuming auth_helper is separate

// Redirect if not authenticated
require_login(); // Using require_login from auth_helper

// Get authenticated user ID
$userId = get_user_id(); // Using get_user_id from auth_helper

// --- Handle Remove Favorite Action (Form POST) ---
// This block processes the form submission for removing a favorite BEFORE displaying the page content
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['remove_favorite_id'])) {
    $movieIdToRemove = filter_var($_POST['remove_favorite_id'], FILTER_VALIDATE_INT);

    if ($movieIdToRemove && $userId) { // Ensure movie ID is valid integer and user is logged in
        // Assuming removeFromFavorites exists in ../includes/config.php and returns true/false
        if (removeFromFavorites($movieIdToRemove, $userId)) {
            $_SESSION['success_message'] = 'Film berhasil dihapus dari favorit.';
        } else {
            $_SESSION['error_message'] = 'Gagal menghapus film dari favorit.';
        }
    } else {
        $_SESSION['error_message'] = 'ID film tidak valid atau pengguna tidak terautentikasi.';
    }

    // Redirect back to the same page using GET request to prevent form resubmission on refresh
    header('Location: index.php');
    exit; // Stop script execution after redirect
}


// --- Data Fetching for Display ---
// Fetch user's favorite movies after handling any potential removal
// Assuming getUserFavorites exists in ../includes/config.php and returns an array of movies
// Each movie object/array should contain 'movie_id', 'title', 'release_date', 'genres' (as array), 'poster_image', 'average_rating'
$movies = getUserFavorites($userId);


// --- Get Messages from Session (for display AFTER redirect) ---
// Retrieve messages that might have been set by the POST handling block
$success_message = isset($_SESSION['success_message']) ? $_SESSION['success_message'] : null;
unset($_SESSION['success_message']); // Clear message after reading
$error_message = isset($_SESSION['error_message']) ? $_SESSION['error_message'] : null;
unset($_SESSION['error_message']); // Clear message after reading

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Favorites</title>
     <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <!-- Use Font Awesome 6.0.0 as in your code -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
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
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
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
                    <input type="text" id="searchInput" placeholder="Search favourites...">
                    <button type="button"><i class="fas fa-search"></i></button> <!-- Changed to type="button" to prevent form submission -->
                </div>
            </div>

             <?php
             // Display success or error messages from session
             if ($success_message): ?>
                <div class="alert success"><?php echo htmlspecialchars($success_message, ENT_QUOTES, 'UTF-8'); ?></div>
            <?php endif; ?>
            <?php if ($error_message): ?>
                <div class="alert error"><?php echo htmlspecialchars($error_message, ENT_QUOTES, 'UTF-8'); ?></div>
            <?php endif; ?>

            <div class="review-grid">
                <?php if (!empty($movies)): ?>
                    <?php foreach ($movies as $movie): ?>
                        <div class="movie-card" data-movie-id="<?php echo htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8'); ?>">
                            <div class="movie-poster" onclick="window.location.href='../review/movie-details.php?id=<?php echo htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8'); ?>'">
                                <!-- Use ?? '' for variables passed to htmlspecialchars -->
                                <!-- Assuming poster_image stores web-accessible path -->
                                <img src="<?php echo htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg', ENT_QUOTES, 'UTF-8'); ?>" alt="<?php echo htmlspecialchars($movie['title'] ?? 'Untitled Movie', ENT_QUOTES, 'UTF-8'); ?>">
                                <div class="movie-actions">
                                     <!-- Form to remove favorite -->
                                     <!-- Use a button with type="submit" inside a form -->
                                     <form action="index.php" method="POST" onsubmit="return confirm('Apakah Anda yakin ingin menghapus film ini dari favorit?');">
                                         <input type="hidden" name="remove_favorite_id" value="<?php echo htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8'); ?>">
                                         <button type="submit" class="action-btn" title="Hapus dari favorit"><i class="fas fa-trash"></i></button>
                                     </form>
                                </div>
                            </div>
                            <div class="movie-details">
                                <!-- Use ?? '' for variables passed to htmlspecialchars -->
                                <h3><?php echo htmlspecialchars($movie['title'] ?? 'Untitled Movie', ENT_QUOTES, 'UTF-8'); ?></h3>
                                <p class="movie-info">
                                    <!-- Use ?? '' for variables passed to htmlspecialchars -->
                                    <?php echo (isset($movie['release_date']) && $movie['release_date']) ? htmlspecialchars((new DateTime($movie['release_date']))->format('Y'), ENT_QUOTES, 'UTF-8') : 'N/A'; ?> |
                                    <?php echo htmlspecialchars(implode(', ', $movie['genres'] ?? []), ENT_QUOTES, 'UTF-8') ?: 'N/A'; ?>
                                </p>
                                <div class="rating">
                                    <div class="stars">
                                        <?php
                                        // Display average rating stars
                                        $average_rating = floatval($movie['average_rating'] ?? '0.0'); // Use raw float value for logic
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
                                    <span class="rating-count">(<?php echo htmlspecialchars($movie['average_rating'] ?? '0.0', ENT_QUOTES, 'UTF-8'); ?>)</span>
                                </div>
                            </div>
                        </div>
                    <?php endforeach; ?>
                <?php endif; ?>
                 <!-- Empty state when NO movies are favorited -->
                 <?php if (empty($movies)): ?>
                     <div class="empty-state full-width">
                         <i class="fas fa-heart-broken"></i>
                         <p>Anda belum menambahkan film ke favorit.</p>
                         <p class="subtitle">Temukan film di bagian <a href="../review/index.php">Review</a> dan tambahkan.</p>
                     </div>
                 <?php endif; ?>
                 <!-- This div is a placeholder for the search empty state -->
                 <!-- JS will show/hide this based on search results -->
                 <div class="empty-state search-empty-state full-width" style="display: none;">
                      <i class="fas fa-search"></i>
                      <p>Film favorit tidak ditemukan.</p>
                      <p class="subtitle">Coba kata kunci pencarian lain.</p>
                 </div>
            </div>
        </main>
    </div>
    <!-- Link to the main JS file which contains search logic and favorite button toggle logic -->
    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
    <script>
        // Specific JS for this page if needed, but much is handled by main.js
        // The search logic is handled by the centralized code in main.js
        // The favorite button (trash icon) submit logic is handled by the form POST
        // The favorite button (heart icon) toggle logic on other pages is in main.js

        // The client-side htmlspecialchars function provided in the original snippet is also removed
        // as escaping is done in PHP before outputting HTML.

        // If you need to customize search behavior specifically for the favorites page
        // beyond what's in main.js, you can add code here.
        // For now, main.js should handle the searchInput and the empty states correctly.
    </script>
</body>
</html>
```

**8. `review/index.php` (Modifikasi Sidebar Link, Popup Data, Empty State)**

Tambahkan link "Manage" dan "Profile" di sidebar, pastikan data yang dilewatkan ke popup lengkap untuk trailer.

```php
<?php
// review/index.php
require_once __DIR__ . '/../config/database.php'; // Adjust path based on your config file location
require_once __DIR__ . '/../includes/auth_helper.php'; // Assuming auth_helper is separate

// Redirect to login if not logged in (Optional: make review page public?)
require_login(); // Using require_login from auth_helper

// Get logged-in user ID for favorite check
$userId = get_user_id(); // Using get_user_id from auth_helper

// Fetch all movies from the database
$movies = getAllMovies(); // Assuming this function exists and returns all movies with genres

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RATE-TALES - Review</title>
     <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Use 6.0.0 -->
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
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
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
                    <input type="text" id="searchInput" placeholder="Search movies...">
                    <button type="button"><i class="fas fa-search"></i></button> <!-- Changed to type="button" -->
                </div>
            </div>
            <div class="review-grid">
                <?php if (!empty($movies)): ?>
                    <?php foreach ($movies as $movie):
                        // Prepare data for display and popup
                        $movieTitle = htmlspecialchars($movie['title'] ?? 'Untitled Movie', ENT_QUOTES, 'UTF-8');
                        $movieId = htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8');
                        $posterImage = htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg', ENT_QUOTES, 'UTF-8');

                        $releaseYear = (isset($movie['release_date']) && $movie['release_date']) ? htmlspecialchars((new DateTime($movie['release_date']))->format('Y'), ENT_QUOTES, 'UTF-8') : 'N/A';
                        $genresString = htmlspecialchars(implode(', ', $movie['genres'] ?? []), ENT_QUOTES, 'UTF-8') ?: 'N/A';
                        $averageRating = htmlspecialchars(getAverageMovieRating($movie['movie_id'] ?? null) ?? '0.0', ENT_QUOTES, 'UTF-8'); // Fetch average rating and handle potential null

                        // Prepare summary and trailer URL for passing to JS popup
                        // Summary needs to be JSON encoded for safe parsing in JS
                        $summaryJson = json_encode($movie['summary'] ?? ''); // JSON encode potential null/string
                        $summaryJsonEscaped = htmlspecialchars($summaryJson, ENT_QUOTES, 'UTF-8'); // Escape the JSON string itself

                        // Determine which trailer URL to use and escape it
                        $trailerUrlToUse = '';
                         if (!empty($movie['trailer_link'])) {
                             $trailerUrlToUse = htmlspecialchars($movie['trailer_link'], ENT_QUOTES, 'UTF-8');
                         } elseif (!empty($movie['trailer_file'])) {
                             $trailerUrlToUse = htmlspecialchars($movie['trailer_file'], ENT_QUOTES, 'UTF-8'); // This is the path
                         }
                         // Pass the selected trailer URL to JS

                        // Check if the movie is favorited by the current user
                        $isFavorited = isFavorite($movie['movie_id'] ?? null, $userId);

                    ?>
                    <div class="movie-card" data-movie-id="<?php echo $movieId; ?>"
                         onclick="showMovieDetailsPopup(
                             <?php echo $movieId; ?>,
                             '<?php echo $movieTitle; ?>',
                             '<?php echo $releaseYear; ?>',
                             '<?php echo $genresString; ?>',
                             '<?php echo $summaryJsonEscaped; ?>',
                             '<?php echo $averageRating; ?>',
                             '<?php echo $trailerUrlToUse; ?>' // Pass trailer URL to popup function
                         )">
                        <div class="movie-poster">
                            <img src="<?php echo $posterImage; ?>" alt="<?php echo $movieTitle; ?>">
                            <div class="movie-actions">
                                <!-- Favorite Button -->
                                <!-- Use classes for JS targeting. Icon class updated by JS. -->
                                <button class="action-btn favorite-btn <?php echo $isFavorited ? 'fas' : 'far'; ?> fa-heart" title="<?php echo $isFavorited ? 'Hapus dari favorit' : 'Tambahkan ke favorit'; ?>"></button>
                                <!-- Optional: Quick Review Button -->
                                <!-- <button class="action-btn review-btn" title="Write a quick review"><i class="fas fa-pen"></i></button> -->
                            </div>
                        </div>
                        <div class="movie-details">
                            <h3><?php echo $movieTitle; ?></h3>
                            <p class="movie-info"><?php echo $releaseYear; ?> | <?php echo $genresString; ?></p>
                            <div class="rating">
                                <div class="stars">
                                    <?php
                                    $rating = floatval($averageRating); // Use fetched average rating string
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
                                <span class="rating-count">(<?php echo $averageRating; ?>)</span>
                            </div>
                        </div>
                    </div>
                    <?php endforeach; ?>
                <?php else: ?>
                     <!-- Empty state when NO movies are available -->
                     <div class="empty-state">
                        <i class="fas fa-film"></i>
                        <p>Film tidak tersedia.</p>
                        <p class="subtitle">Film akan muncul di sini setelah diunggah.</p>
                    </div>
                <?php endif; ?>
                 <!-- Placeholder for search empty state -->
                 <div class="empty-state search-empty-state full-width" style="display: none;">
                      <i class="fas fa-search"></i>
                      <p>Film tidak ditemukan.</p>
                      <p class="subtitle">Coba kata kunci pencarian lain.</p>
                 </div>
            </div>
        </main>
    </div>

    <!-- Movie Details Popup -->
    <div id="movie-details-popup" class="movie-details-popup">
        <div class="popup-content">
            <div class="popup-header">
                <h2 id="popup-title"></h2>
                <div class="popup-rating">
                     <!-- Stars for user rating input (client-side visual only from popup) -->
                    <div class="rating-stars">
                        <!-- Stars will be dynamically added/managed by JS -->
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
                 <!-- Favorite Button in Popup -->
                 <!-- Needs data-movie-id attribute, set by JS -->
                 <button class="action-button add-favorite-popup-btn favorite-btn"><i class="fas fa-heart"></i> Add to Favorites</button>
                 <!-- Watch Trailer Button in Popup -->
                 <!-- Needs data-trailer-url attribute, set by JS -->
                  <button class="action-button watch-trailer"><i class="fas fa-play"></i> Watch Trailer</button>
                 <!-- Read More Button in Popup -->
                 <button class="action-button read-more-btn" onclick="openMovieDetails()"><i class="fas fa-book-open"></i> Read More</button>
            </div>
        </div>
    </div>

    <!-- Trailer Modal (Keep outside popup) -->
    <div id="trailer-modal" class="trailer-modal">
        <div class="trailer-content">
            <span class="close-trailer" onclick="closeTrailer()">&times;</span>
            <div class="video-container">
                <!-- Iframe for YouTube -->
                 <iframe id="trailer-iframe" src="" frameborder="0" allowfullscreen></iframe>
                 <!-- Video tag for local files -->
                 <video id="trailer-video" controls style="display: none;"></video> <!-- Initially hidden -->
            </div>
        </div>
    </div>


    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
    <script>
        // Specific JS for this page if needed.
        // showMovieDetailsPopup, openMovieDetails functions are now in main.js
        // Favorite button toggle logic is in main.js
        // Search logic is in main.js

        // Data for popup is passed directly from PHP loop

        // Add event listeners for buttons inside the popup after it's populated
        // The favorite button listener is handled by the general `.favorite-btn` selector in main.js
        // The watch trailer button listener needs to be set up to call window.playTrailer
        // This is now done in the showMovieDetailsPopup function in main.js
    </script>
</body>
</html>
```

**9. `review/movie-details.php` (Modifikasi Sidebar Link, Trailer Modal, Comments)**

Tambahkan link "Manage" dan "Profile" di sidebar, perbaiki HTML modal trailer untuk mendukung file lokal, dan pastikan tampilan komentar benar.

```php
<?php
// review/movie-details.php
require_once __DIR__ . '/../config/database.php'; // Adjust path based on your config file location
require_once __DIR__ . '/../includes/auth_helper.php'; // Assuming auth_helper is separate

// Redirect to login if not logged in (Optional: make details page public?)
require_login(); // Using require_login from auth_helper

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
    $movie = getMovieById($movieId); // Assuming getMovieById exists and fetches genres
    if ($movie) {
        $reviews = getMovieReviews($movieId); // Fetch reviews for this movie
        $averageRating = getAverageMovieRating($movieId); // Fetch average rating
        $isFavorited = isFavorite($movieId, $userId); // Check if favorite

        // Determine which trailer to use: trailer_link (YouTube) or trailer_file (local)
        if (!empty($movie['trailer_link'])) {
            $trailerUrlToUse = htmlspecialchars($movie['trailer_link'], ENT_QUOTES, 'UTF-8');
        } elseif (!empty($movie['trailer_file'])) {
             // Use the local file path (should be web-accessible /uploads/trailers/...)
             $trailerUrlToUse = htmlspecialchars($movie['trailer_file'], ENT_QUOTES, 'UTF-8');
        }

    }
}

// Handle movie not found
if (!$movie) {
    // Display an error or redirect
    $_SESSION['error_message'] = 'Film tidak ditemukan.'; // Optional message
    header('Location: index.php'); // Redirect back to review list
    exit;
}

// Prepare data for display
$movieTitle = htmlspecialchars($movie['title'] ?? 'Untitled Movie', ENT_QUOTES, 'UTF-8');
$posterImage = htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg', ENT_QUOTES, 'UTF-8');
$releaseYear = (isset($movie['release_date']) && $movie['release_date']) ? htmlspecialchars((new DateTime($movie['release_date']))->format('Y'), ENT_QUOTES, 'UTF-8') : 'N/A';
$durationHours = htmlspecialchars($movie['duration_hours'] ?? 0, ENT_QUOTES, 'UTF-8');
$durationMinutes = htmlspecialchars($movie['duration_minutes'] ?? 0, ENT_QUOTES, 'UTF-8');
$ageRating = htmlspecialchars($movie['age_rating'] ?? 'N/A', ENT_QUOTES, 'UTF-8');
$genresString = htmlspecialchars(implode(', ', $movie['genres'] ?? []), ENT_QUOTES, 'UTF-8') ?: 'N/A';
$summary = htmlspecialchars($movie['summary'] ?? 'No summary available.', ENT_QUOTES, 'UTF-8'); // Escape summary


// --- Handle Comment Submission (AJAX Endpoint) ---
// The form will post to review/submit_review.php, handled by JS in main.js

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?php echo $movieTitle; ?> - RATE-TALES</title>
     <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Use 6.0.0 -->
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
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
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
                        <img id="movie-poster" src="<?php echo $posterImage; ?>" alt="<?php echo $movieTitle; ?>">
                    </div>
                    <div class="movie-info-large">
                        <h1 id="movie-title" class="movie-title-large"><?php echo $movieTitle; ?></h1>
                        <p id="movie-meta" class="movie-meta">
                            <?php echo $releaseYear; ?> |
                            <?php echo $genresString; ?> |
                            <?php echo $durationHours . 'h ' . $durationMinutes . 'm'; ?> |
                            Rated: <?php echo $ageRating; ?>
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
                            <span id="movie-rating"><?php echo $averageRating; ?>/5</span>
                        </div>
                        <p id="movie-description" class="movie-description"><?php echo $summary; ?></p>
                        <div class="action-buttons">
                            <!-- Watch Trailer Button -->
                            <!-- Call playTrailer JS function with the determined trailer URL -->
                            <button class="action-button watch-trailer" onclick="playTrailer('<?php echo $trailerUrlToUse; ?>')">
                                <i class="fas fa-play"></i>
                                <span>Watch Trailer</span>
                            </button>
                            <!-- Favorite Button -->
                            <!-- Use classes for JS targeting. Icon class updated by JS. Needs movie-id -->
                            <button class="action-button favorite-btn" data-movie-id="<?php echo htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8'); ?>">
                                <i class="fas <?php echo $isFavorited ? 'fa-heart' : 'far fa-heart'; ?>"></i> <!-- Icon reflects favorite status -->
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
                              <div class="stars" id="user-rating-stars" data-movie-id="<?php echo htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8'); ?>">
                                 <i class="far fa-star" data-rating="1"></i>
                                 <i class="far fa-star" data-rating="2"></i>
                                 <i class="far fa-star" data-rating="3"></i>
                                 <i class="far fa-star" data-rating="4"></i>
                                 <i class="far fa-star" data-rating="5"></i>
                             </div>
                         </div>
                         <!-- Comment Form -->
                         <!-- Form will be submitted via AJAX by JS in main.js -->
                         <form id="comment-form">
                             <input type="hidden" name="movie_id" value="<?php echo htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8'); ?>">
                             <input type="hidden" name="rating" id="submitted-rating" value="0"> <!-- Hidden input for rating value set by JS -->
                             <textarea name="comment" class="comment-input" placeholder="Write a comment..." required></textarea>
                              <button type="submit" class="action-button" style="background-color: #00ffff; color: #1a1a1a;">Submit Review</button>
                         </form>
                    </div>


                    <div class="comment-list">
                        <?php if (!empty($reviews)): ?>
                            <?php foreach ($reviews as $review): ?>
                            <div class="comment">
                                <div class="comment-header">
                                     <!-- User profile image next to username -->
                                     <!-- Assuming profile_image stores web-accessible path -->
                                     <img src="<?php echo htmlspecialchars($review['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($review['username'] ?? 'User') . '&background=random') ; ?>" alt="User Avatar" style="width: 30px; height: 30px; border-radius: 50%; margin-right: 10px; vertical-align: middle;">
                                    <strong><?php echo htmlspecialchars($review['username'] ?? 'Unknown User', ENT_QUOTES, 'UTF-8'); ?></strong>
                                    <div class="comment-actions">
                                         <!-- Display rating for this specific review -->
                                          <span style="color: #ffd700; margin-right: 10px;">
                                            <?php
                                             $reviewRating = floatval($review['rating'] ?? '0.0');
                                             for ($i = 1; $i <= 5; $i++) {
                                                 if ($i <= $reviewRating) {
                                                     echo '<i class="fas fa-star" style="font-size: 0.9em;"></i>';
                                                 } else if ($i - 0.5 <= $reviewRating) {
                                                     echo '<i class="fas fa-star-half-alt" style="font-size: 0.9em;"></i>';
                                                 } else {
                                                     echo '<i class="far fa-star" style="font-size: 0.9em;"></i>';
                                                 }
                                             }
                                             echo " (" . htmlspecialchars($review['rating'] ?? '0.0', ENT_QUOTES, 'UTF-8') . ")";
                                            ?>
                                         </span>
                                        <!-- Placeholder actions -->
                                        <!-- <i class="fas fa-thumbs-up"></i> -->
                                        <!-- <i class="fas fa-thumbs-down"></i> -->
                                        <!-- <i class="fas fa-reply"></i> -->
                                    </div>
                                </div>
                                <!-- Escape comment text and preserve newlines -->
                                <p><?php echo nl2br(htmlspecialchars($review['comment'] ?? '', ENT_QUOTES, 'UTF-8')); ?></p>
                                 <div class="comment-date" style="font-size: 0.8em; color: #666; margin-top: 5px;">
                                     <?php echo (isset($review['created_at']) && $review['created_at']) ? htmlspecialchars(date('Y-m-d H:i', strtotime($review['created_at'])), ENT_QUOTES, 'UTF-8') : 'N/A'; ?>
                                 </div>
                            </div>
                            <?php endforeach; ?>
                        <?php else: ?>
                            <!-- Empty state for comments -->
                            <div class="empty-state" style="padding: 20px;">
                                <i class="fas fa-comment"></i>
                                <p>Belum ada ulasan.</p>
                                <p class="subtitle">Jadilah yang pertama memberikan ulasan!</p>
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
                <!-- Iframe for YouTube -->
                 <iframe id="trailer-iframe" src="" frameborder="0" allowfullscreen></iframe>
                 <!-- Video tag for local files -->
                 <video id="trailer-video" controls style="display: none;"></video> <!-- Initially hidden -->
            </div>
        </div>
    </div>

    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
     <script>
         // Specific JS for this page if needed.
         // playTrailer, closeTrailer, etc. are now in main.js
         // User rating input and comment form submission logic are in main.js
         // Favorite button logic is in main.js
     </script>
</body>
</html>
```

**10. `manage/index.php` (Modifikasi Sidebar Link dan Empty State)**

Tambahkan link "Profile" di sidebar, perbaiki penanganan empty state jika tidak ada film diunggah pengguna.

```php
<?php
// manage/index.php
require_once __DIR__ . '/../config/database.php'; // Adjust path based on your config file location
require_once __DIR__ . '/../includes/auth_helper.php'; // Assuming auth_helper is separate

// Redirect to login if not logged in
require_login(); // Using require_login from auth_helper

$userId = get_user_id(); // Using get_user_id from auth_helper

// Fetch movies uploaded by the current user
$uploadedMovies = getMoviesByUserId($userId); // Assuming this function exists

?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Manage Movies - RatingTales</title>
     <link rel="icon" type="image/png" href="../gambar/untitled142_20250310223718.png"> <!-- Adjust path -->
    <link rel="stylesheet" href="../css/style.css"> <!-- Adjusted CSS path -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Use 6.0.0 -->
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
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
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
                    <input type="text" id="searchInput" placeholder="Search movies...">
                    <button type="button"><i class="fas fa-search"></i></button> <!-- Changed to type="button" -->
                </div>
            </div>

            <!-- Movies Grid -->
            <div class="movies-grid">
                <?php if (empty($uploadedMovies)): ?>
                    <!-- Empty state when NO movies are uploaded -->
                    <div class="empty-state full-width">
                        <i class="fas fa-film"></i>
                        <p>Anda belum mengunggah film apapun.</p>
                        <p class="subtitle">Mulai tambahkan film Anda dengan mengklik tombol unggah di bawah.</p>
                    </div>
                <?php else: ?>
                     <?php foreach ($uploadedMovies as $movie):
                          // Prepare data for display
                          $movieTitle = htmlspecialchars($movie['title'] ?? 'Untitled Movie', ENT_QUOTES, 'UTF-8');
                          $movieId = htmlspecialchars($movie['movie_id'] ?? '', ENT_QUOTES, 'UTF-8');
                          $posterImage = htmlspecialchars($movie['poster_image'] ?? '../gambar/placeholder.jpg', ENT_QUOTES, 'UTF-8');
                          $releaseYear = (isset($movie['release_date']) && $movie['release_date']) ? htmlspecialchars((new DateTime($movie['release_date']))->format('Y'), ENT_QUOTES, 'UTF-8') : 'N/A';
                          $genresString = htmlspecialchars(implode(', ', $movie['genres'] ?? []), ENT_QUOTES, 'UTF-8') ?: 'N/A';
                          $averageRating = htmlspecialchars(getAverageMovieRating($movie['movie_id'] ?? null) ?? '0.0', ENT_QUOTES, 'UTF-8'); // Fetch average rating

                     ?>
                     <div class="movie-card" data-movie-id="<?php echo $movieId; ?>">
                          <div class="movie-poster">
                              <img src="<?php echo $posterImage; ?>" alt="<?php echo $movieTitle; ?>">
                              <div class="movie-actions">
                                   <!-- Edit Button Placeholder -->
                                   <button class="action-btn edit-btn" title="Edit Movie"><i class="fas fa-edit"></i></button>
                                   <!-- Delete Button Placeholder -->
                                   <!-- Implement delete via form or AJAX -->
                                    <!-- Example using a simple form -->
                                   <form action="delete_movie.php" method="POST" onsubmit="return confirm('Apakah Anda yakin ingin menghapus film ini?');" style="display:inline;">
                                        <input type="hidden" name="movie_id_to_delete" value="<?php echo $movieId; ?>">
                                        <button type="submit" class="action-btn delete-btn" title="Hapus Film"><i class="fas fa-trash"></i></button>
                                   </form>
                              </div>
                         </div>
                         <div class="movie-details">
                             <h3><?php echo $movieTitle; ?></h3>
                             <p class="movie-info"><?php echo $releaseYear; ?> | <?php echo $genresString; ?></p>
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
                                  <span class="rating-count">(<?php echo $averageRating; ?>)</span>
                              </div>
                         </div>
                     </div>
                     <?php endforeach; ?>
                <?php endif; ?>
                 <!-- Placeholder for search empty state -->
                 <!-- JS will show/hide this based on search results -->
                 <div class="empty-state search-empty-state full-width" style="display: none;">
                      <i class="fas fa-search"></i>
                      <p>Film tidak ditemukan.</p>
                      <p class="subtitle">Coba kata kunci pencarian lain.</p>
                 </div>
            </div>

            <!-- Action Buttons -->
            <div class="action-buttons">
                <!-- Edit All Button - Functionality not implemented -->
                <button class="edit-all-btn" title="Edit All Movies">
                    <i class="fas fa-edit"></i>
                </button>
                <!-- Upload New Movie Button -->
                <a href="upload.php" class="upload-btn" title="Upload New Movie">
                    <i class="fas fa-plus"></i>
                </a>
            </div>
        </main>
    </div>
    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
    <script>
         // Specific JS for manage page actions (edit/delete) if needed
         document.addEventListener('DOMContentLoaded', function() {
             // Delete button functionality - Handled by form submit now, not AJAX
             // If you prefer AJAX, you would remove the <form> tags around the button
             // and add an event listener here that sends an AJAX request to delete_movie.php
             // and removes the card on success.

              // Edit button functionality (requires dedicated edit page or modal)
             document.querySelectorAll('.edit-btn').forEach(button => {
                 button.addEventListener('click', function(e) {
                      e.stopPropagation(); // Prevent parent click
                      e.preventDefault();

                      const movieId = this.closest('.movie-card').getAttribute('data-movie-id');
                      if(movieId) {
                           // TODO: Redirect to an edit page with movie ID, e.g., window.location.href = 'edit_movie.php?id=' + movieId;
                           console.log("Editing movie ID:", movieId);
                           alert("Fitur edit belum diimplementasikan."); // Placeholder
                      } else {
                           console.error("Movie ID not found for edit button.");
                           alert("Tidak dapat mengedit film (ID film tidak ditemukan).");
                      }
                 });
             });

             // Ensure search input works - handled by main.js
             // Ensure empty states for search are handled - handled by main.js
              // Ensure initial empty state is hidden if there are movies - handled by main.js by checking .movies-grid children
         });
    </script>
</body>
</html>
```

**11. `acc_page/index.php` (Modifikasi Sidebar Link dan Tampilan Data User)**

Tambahkan link "Manage" di sidebar dan pastikan data user diambil dan ditampilkan dengan benar.

```php
<?php
// acc_page/index.php
require_once __DIR__ . '/../config/database.php'; // Adjust path based on your config file location
require_once __DIR__ . '/../includes/auth_helper.php'; // Assuming auth_helper is separate

// Redirect to login if not logged in
require_login(); // Using require_login from auth_helper

$userId = get_user_id(); // Using get_user_id from auth_helper
$user = getUserById($userId); // Assuming getUserById fetches full_name, username, profile_image, bio

// Handle case where user somehow isn't found (shouldn't happen after require_login)
if (!$user) {
     error_log("Logged in user ID {$userId} not found in database.");
     // Redirect to logout or error page
     header('Location: ../autentikasi/logout.php');
     exit;
}

// Prepare user data for display
$displayName = htmlspecialchars($user['full_name'] ?? $user['username'] ?? 'User', ENT_QUOTES, 'UTF-8'); // Use full_name, fallback to username, then 'User'
$username = htmlspecialchars($user['username'] ?? 'User', ENT_QUOTES, 'UTF-8');
// Assuming profile_image is stored as a path relative to webroot /uploads/profiles/ or is null
$profileImagePath = htmlspecialchars($user['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($username) . '&background=random', ENT_QUOTES, 'UTF-8');
// Bio text - handle null and display placeholder if empty after trim
$bioText = htmlspecialchars($user['bio'] ?? '', ENT_QUOTES, 'UTF-8');
$displayBio = $bioText ? $bioText : 'Click to add bio...';


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
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Use 6.0.0 -->
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
                 <li class="active"><a href="#"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
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
                        <img src="<?php echo $profileImagePath; ?>" alt="Profile Picture">
                        <!-- Optional: Add an icon/button/form to change profile picture (requires implementation) -->
                         <!-- <i class="fas fa-camera change-photo-icon"></i> -->
                    </div>
                    <div class="profile-details">
                        <h1>
                            <!-- Display Name (Full Name from DB) -->
                            <span id="displayName"><?php echo $displayName; ?></span>
                            <!-- Pass DB field name to JS function -->
                            <i class="fas fa-pen edit-icon" onclick="toggleEdit('full_name')"></i>
                        </h1>
                        <p class="username">
                            @<span id="username"><?php echo $username; ?></span>
                            <!-- Pass DB field name to JS function -->
                            <i class="fas fa-pen edit-icon" onclick="toggleEdit('username')"></i>
                        </p>
                         <!-- Optional: Display Email, Age, Gender -->
                         <!-- <p>Email: <?php echo htmlspecialchars($user['email'] ?? 'N/A', ENT_QUOTES, 'UTF-8'); ?></p> -->
                         <!-- <p>Usia: <?php echo htmlspecialchars($user['age'] ?? 'N/A', ENT_QUOTES, 'UTF-8'); ?></p> -->
                         <!-- <p>Jenis Kelamin: <?php echo htmlspecialchars($user['gender'] ?? 'N/A', ENT_QUOTES, 'UTF-8'); ?></p> -->
                         <!-- <p>Bergabung Sejak: <?php echo (isset($user['created_at']) && $user['created_at']) ? htmlspecialchars(date('M Y', strtotime($user['created_at'])), ENT_QUOTES, 'UTF-8') : 'N/A'; ?></p> -->
                    </div>
                </div>
                <div class="about-me">
                    <h2>ABOUT ME:</h2>
                    <!-- Pass DB field name to JS function -->
                    <div class="about-content" id="bio" onclick="toggleEdit('bio')">
                        <?php echo $displayBio; ?> <!-- Bio text is already HTML escaped -->
                    </div>
                </div>
            </div>

             <!-- Removed "My Posts" section as it's not in the database schema -->

        </main>
    </div>
    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
    <script>
        // toggleEdit function is now in js/main.js
        // It needs to send AJAX calls to acc_page/update_profile.php (needs implementation)

        // Profile picture change is a separate feature not implemented here.
    </script>
</body>
</html>
```

**12. `manage/upload.php` (Modifikasi Sidebar Link)**

Tambahkan link "Profile" di sidebar.

```php
<?php
// manage/upload.php
require_once __DIR__ . '/../config/database.php'; // Adjust path based on your config file location
require_once __DIR__ . '/../includes/auth_helper.php'; // Assuming auth_helper is separate

// Redirect to login if not logged in
require_login(); // Using require_login from auth_helper

$userId = get_user_id(); // Using get_user_id from auth_helper

// Check for error/success messages and old form data from session set by upload_handler.php
$error_message = null;
$success_message = null;
$old_form_data = [];

if (isset($_SESSION['error'])) {
    $error_message = $_SESSION['error'];
    unset($_SESSION['error']);
}
// Success message is usually for the index page after redirect from handler,
// but keep it here for consistency if needed.
// if (isset($_SESSION['success'])) {
//     $success_message = $_SESSION['success'];
//     unset($_SESSION['success']);
// }
if (isset($_SESSION['old_form_data'])) {
    $old_form_data = $_SESSION['old_form_data'];
    unset($_SESSION['old_form_data']);
}

// Helper function to get old value for form fields
function get_old_value($field_name, $default = '') {
    global $old_form_data;
    return htmlspecialchars($old_form_data[$field_name] ?? $default, ENT_QUOTES, 'UTF-8');
}

// Helper function to check if a checkbox value was old_checked
function is_old_checked($field_name, $value) {
    global $old_form_data;
    // Check if the field exists and if the value is in the array for that field name
    return isset($old_form_data[$field_name]) && is_array($old_form_data[$field_name]) && in_array($value, $old_form_data[$field_name]);
}

// Helper function to check if a select option was old_selected
function is_old_selected($field_name, $value) {
     global $old_form_data;
     // Check if the field exists and its value matches
     return isset($old_form_data[$field_name]) && ($old_form_data[$field_name] === $value);
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
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Use 6.0.0 -->
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
                 <li><a href="../acc_page/index.php"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
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
                     <div class="alert error" style="margin-bottom: 20px;"><?php echo htmlspecialchars($error_message, ENT_QUOTES, 'UTF-8'); ?></div>
                 <?php endif; ?>
                 <?php if ($success_message): ?>
                     <div class="alert success" style="margin-bottom: 20px;"><?php echo htmlspecialchars($success_message, ENT_QUOTES, 'UTF-8'); ?></div>
                 <?php endif; ?>

                <!-- FORM ACTION POINTS TO HANDLER -->
                <form class="upload-form" action="upload_handler.php" method="post" enctype="multipart/form-data">
                    <div class="form-layout">
                        <div class="form-main">
                            <div class="form-group">
                                <label for="movie-title">Movie Title</label>
                                <!-- Populate with old value -->
                                <input type="text" id="movie-title" name="movie-title" required value="<?php echo get_old_value('movie-title'); ?>">
                            </div>

                            <div class="form-group">
                                <label for="movie-summary">Movie Summary</label>
                                <!-- Populate with old value -->
                                <textarea id="movie-summary" name="movie-summary" rows="4" required><?php echo get_old_value('movie-summary'); ?></textarea>
                            </div>

                            <div class="form-group">
                                <label>Genre</label>
                                <div class="genre-options">
                                    <?php
                                    // Dynamically generate genre options based on ENUM values from DB if possible,
                                    // or list them manually matching the ENUM ('action', 'adventure', 'comedy', 'drama', 'horror')
                                    $genresOptions = ['action', 'adventure', 'comedy', 'drama', 'horror'];
                                    foreach ($genresOptions as $genreOption):
                                        // Check if genre was old_checked
                                        $isChecked = is_old_checked('genre', $genreOption);
                                    ?>
                                    <label class="checkbox-label">
                                        <input type="checkbox" name="genre[]" value="<?php echo htmlspecialchars($genreOption, ENT_QUOTES, 'UTF-8'); ?>" <?php echo $isChecked ? 'checked' : ''; ?>>
                                        <span><?php echo htmlspecialchars(ucfirst($genreOption), ENT_QUOTES, 'UTF-8'); ?></span> <!-- Display capitalized -->
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
                                ?>
                                <select id="age-rating" name="age-rating" required>
                                    <option value="">Select age rating</option>
                                    <?php foreach ($ageRatings as $value => $label): ?>
                                         <!-- Check if option was old_selected -->
                                         <option value="<?php echo htmlspecialchars($value, ENT_QUOTES, 'UTF-8'); ?>" <?php echo is_old_selected('age-rating', $value) ? 'selected' : ''; ?>>
                                              <?php echo htmlspecialchars($label, ENT_QUOTES, 'UTF-8'); ?>
                                         </option>
                                    <?php endforeach; ?>
                                </select>
                            </div>

                            <div class="form-group">
                                <label for="movie-trailer">Movie Trailer</label>
                                <div class="trailer-input">
                                    <!-- Populate with old value -->
                                    <input type="text" id="trailer-link" name="trailer-link" placeholder="Enter YouTube video URL" value="<?php echo get_old_value('trailer-link'); ?>">
                                    <span class="trailer-note">* Paste YouTube video URL</span>
                                </div>
                                <div class="trailer-upload">
                                    <!-- File inputs cannot be pre-filled for security reasons -->
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
                                    <!-- File inputs cannot be pre-filled for security reasons -->
                                    <input type="file" id="movie-poster" name="movie-poster" accept="image/*" required>
                                </div>
                                <!-- Optional: Display preview after file selection -->
                                 <img id="poster-preview" src="#" alt="Poster Preview" style="display: none; max-width: 100%; height: auto; margin-top: 10px; border-radius: 8px;">
                            </div>

                            <div class="advanced-settings form-section"> <!-- Add form-section class for spacing -->
                                <h3>Advanced Settings</h3>
                                <div class="form-group">
                                    <label for="release-date">Release Date</label>
                                     <!-- Populate with old value -->
                                    <input type="date" id="release-date" name="release-date" required value="<?php echo get_old_value('release-date'); ?>">
                                </div>

                                <div class="form-group">
                                    <label for="duration-hours">Film Duration</label>
                                    <div class="duration-inputs">
                                        <div class="duration-field">
                                             <!-- Populate with old value -->
                                            <input type="number" id="duration-hours" name="duration-hours" min="0" placeholder="Hours" required value="<?php echo get_old_value('duration-hours'); ?>">
                                            <span>Hours</span>
                                        </div>
                                        <div class="duration-field">
                                             <!-- Populate with old value -->
                                            <input type="number" id="duration-minutes" name="duration-minutes" min="0" max="59" placeholder="Minutes" required value="<?php echo get_old_value('duration-minutes'); ?>">
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
                 const trailerUploadContainer = trailerFileInput.closest('.trailer-upload');
                 const trailerLinkContainer = trailerLinkInput.closest('.trailer-input');

                 function updateTrailerInputsState() {
                      if (trailerLinkInput.value.trim() !== '') {
                          trailerFileInput.disabled = true;
                          if (trailerUploadContainer) trailerUploadContainer.style.opacity = 0.5;
                      } else if (trailerFileInput.files.length > 0) {
                           trailerLinkInput.disabled = true;
                           if (trailerLinkContainer) trailerLinkContainer.style.opacity = 0.5;
                      } else {
                          trailerFileInput.disabled = false;
                          trailerLinkInput.disabled = false;
                           if (trailerUploadContainer) trailerUploadContainer.style.opacity = 1;
                           if (trailerLinkContainer) trailerLinkContainer.style.opacity = 1;
                      }
                 }

                 trailerLinkInput.addEventListener('input', updateTrailerInputsState);
                 trailerFileInput.addEventListener('change', updateTrailerInputsState);

                 // Initial state check on page load (important for old data)
                 updateTrailerInputsState();
             }

         });
     </script>
</body>
</html>
```

**13. `acc_page/index.php` (Modifikasi Sidebar Link dan Tampilan Gambar Profil)**

Tambahkan link "Manage" di sidebar, pastikan gambar profil ditampilkan dengan benar (handle null).

```php
<?php
// acc_page/index.php
require_once __DIR__ . '/../config/database.php'; // Adjust path based on your config file location
require_once __DIR__ . '/../includes/auth_helper.php'; // Assuming auth_helper is separate

// Redirect to login if not logged in
require_login(); // Using require_login from auth_helper

$userId = get_user_id(); // Using get_user_id from auth_helper
$user = getUserById($userId); // Assuming getUserById fetches full_name, username, profile_image, bio

// Handle case where user somehow isn't found (shouldn't happen after require_login)
if (!$user) {
     error_log("Logged in user ID {$userId} not found in database.");
     // Redirect to logout or error page
     header('Location: ../autentikasi/logout.php');
     exit;
}

// Prepare user data for display
$displayName = htmlspecialchars($user['full_name'] ?? $user['username'] ?? 'User', ENT_QUOTES, 'UTF-8'); // Use full_name, fallback to username, then 'User'
$username = htmlspecialchars($user['username'] ?? 'User', ENT_QUOTES, 'UTF-8');
// Assuming profile_image is stored as a path relative to webroot /uploads/profiles/ or is null
$profileImagePath = htmlspecialchars($user['profile_image'] ?? 'https://ui-avatars.com/api/?name=' . urlencode($username) . '&background=random', ENT_QUOTES, 'UTF-8');
// Bio text - handle null and display placeholder if empty after trim
$bioText = htmlspecialchars($user['bio'] ?? '', ENT_QUOTES, 'UTF-8');
$displayBio = $bioText ? $bioText : 'Click to add bio...';


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
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css"> <!-- Use 6.0.0 -->
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
                 <li class="active"><a href="#"><i class="fas fa-user"></i> <span>Profile</span></a></li> <!-- Added Profile link -->
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
                        <img src="<?php echo $profileImagePath; ?>" alt="Profile Picture">
                        <!-- Optional: Add an icon/button/form to change profile picture (requires implementation) -->
                         <!-- <i class="fas fa-camera change-photo-icon"></i> -->
                    </div>
                    <div class="profile-details">
                        <h1>
                            <!-- Display Name (Full Name from DB) -->
                            <span id="displayName"><?php echo $displayName; ?></span>
                            <!-- Pass DB field name to JS function -->
                            <i class="fas fa-pen edit-icon" onclick="toggleEdit('full_name')"></i>
                        </h1>
                        <p class="username">
                            @<span id="username"><?php echo $username; ?></span>
                            <!-- Pass DB field name to JS function -->
                            <i class="fas fa-pen edit-icon" onclick="toggleEdit('username')"></i>
                        </p>
                         <!-- Optional: Display Email, Age, Gender -->
                         <!-- <p>Email: <?php echo htmlspecialchars($user['email'] ?? 'N/A', ENT_QUOTES, 'UTF-8'); ?></p> -->
                         <!-- <p>Usia: <?php echo htmlspecialchars($user['age'] ?? 'N/A', ENT_QUOTES, 'UTF-8'); ?></p> -->
                         <!-- <p>Jenis Kelamin: <?php echo htmlspecialchars($user['gender'] ?? 'N/A', ENT_QUOTES, 'UTF-8'); ?></p> -->
                         <!-- <p>Bergabung Sejak: <?php echo (isset($user['created_at']) && $user['created_at']) ? htmlspecialchars(date('M Y', strtotime($user['created_at'])), ENT_QUOTES, 'UTF-8') : 'N/A'; ?></p> -->
                    </div>
                </div>
                <div class="about-me">
                    <h2>ABOUT ME:</h2>
                    <!-- Pass DB field name to JS function -->
                    <div class="about-content" id="bio" onclick="toggleEdit('bio')">
                        <?php echo $displayBio; ?> <!-- Bio text is already HTML escaped -->
                    </div>
                </div>
            </div>

             <!-- Removed "My Posts" section as it's not in the database schema -->

        </main>
    </div>
    <script src="../js/main.js"></script> <!-- Adjusted JS path -->
    <script>
        // toggleEdit function is now in js/main.js
        // It needs to send AJAX calls to acc_page/update_profile.php (needs implementation)

        // Profile picture change is a separate feature not implemented here.
    </script>
</body>
</html>
```

**File-file Lain yang Tidak Perlu Diubah (tetap gunakan versi terakhir yang kita sepakati):**

*   `autentikasi/form-login.php` (Sudah tanpa CAPTCHA/Google)
*   `autentikasi/form-register.php` (Sudah tanpa CAPTCHA/Google)
*   `autentikasi/logout.php`
*   `manage/upload_handler.php`
*   `review/submit_review.php`
*   `favorite/toggle_favorite.php` (Jika Anda menggunakannya sebagai endpoint AJAX untuk tombol hati di halaman lain)
*   `acc_page/update_profile.php` (Jika Anda sudah mengimplementasikan endpoint AJAX ini)
*   `index.php` (File redirect utama di root)
*   File-file endpoint delete/edit film di `manage/` jika Anda sudah membuatnya (misal `delete_movie.php`).

**Langkah-langkah Implementasi:**

1.  Pastikan Anda memiliki semua file yang disebutkan dalam daftar di atas (kecuali yang sengaja dihapus) dengan kode lengkap sesuai dengan versi terakhir yang kita diskusikan.
2.  Ganti kode di `beranda/index.php`, `favorite/index.php`, `review/index.php`, `review/movie-details.php`, `manage/index.php`, `manage/upload.php`, dan `acc_page/index.php` dengan kode lengkap yang saya berikan di atas.
3.  Pastikan `config/database.php` dan `includes/auth_helper.php` Anda benar dan berada di lokasi yang tepat.
4.  Pastikan file `js/main.js` Anda sudah di-update dengan fungsi `playTrailer` yang mendukung video lokal dan YouTube, serta menangani logika pencarian dan favorite button toggle.
5.  Pastikan skema tabel `users` dan `movies` Anda di database sesuai dengan kolom yang digunakan dalam query PHP. Ini sangat penting untuk mencegah error database.
6.  Pastikan folder `uploads/posters/` dan `uploads/trailers/` ada dan memiliki izin tulis untuk server web Anda.
7.  Perbaiki error 403 dengan memeriksa izin file dan folder Anda.

Setelah menerapkan semua perubahan ini, poin-poin "cacat" yang Anda sebutkan seharusnya sudah teratasi berdasarkan fungsionalitas yang ada, kecuali untuk fitur mengganti gambar profil yang memerlukan implementasi tambahan.
