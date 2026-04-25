import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.*;
import java.util.regex.Pattern;
// ============================================================
// CinéBook — Cinema Booking System
// Improvements over original:
// • Password validation (1+ uppercase, 1+ digit, 8+ chars)
// • Email validation (proper regex)
// • Real movies with PHP prices
// • Movie selection & watch flow
// • Wallet system (top-up, balance check)
// • 5 % service fee + total calculation
// • ASCII ticket / QR code on purchase
// ============================================================
public class Main {
 // ── Services & state ────────────────────────────────────
 private static final UserService userService = new UserService();
 private static final MovieService movieService = new MovieService();
 private static final Scanner scanner = new Scanner(System.in);
 private static User currentUser = null;
 // ── Formatting helpers ──────────────────────────────────
 private static final String LINE = "═".repeat(52);
 private static final String THIN = "─".repeat(52);
 public static void main(String[] args) {
 printBanner();
 while (true) {
 if (currentUser == null) {
 showGuestMenu();
 } else if (currentUser.getRole() == User.UserRole.ADMIN) {
 showAdminMenu();
 } else {
 showUserMenu();
 }
 }
 }
 // ── Menus ───────────────────────────────────────────────
 private static void showGuestMenu() {
 System.out.println("\n" + LINE);
 System.out.println(" GUEST MENU");
 System.out.println(LINE);
 System.out.println(" 1. Login");
 System.out.println(" 2. Register");
 System.out.println(" 3. Exit");
 System.out.println(LINE);
 System.out.print(" Choose: ");
 switch (scanner.nextLine().trim()) {
 case "1" -> loginUser();
 case "2" -> registerNewUser();
 case "3" -> { System.out.println("\n Goodbye! \n"); scanner.close(); System.exit(0);
}
 default -> System.out.println(" [!] Invalid option.");
 }
 }
 private static void showUserMenu() {
 System.out.println("\n" + LINE);
 System.out.printf(" USER MENU · %s%n", currentUser.getUsername());
 System.out.printf(" Wallet: ₱%.2f%n", currentUser.getBalance());
 System.out.println(LINE);
 System.out.println(" 1. Browse & Watch Movies");
 System.out.println(" 2. My Tickets");
 System.out.println(" 3. Top Up Wallet");
 System.out.println(" 4. Profile");
 System.out.println(" 5. Logout");
 System.out.println(LINE);
 System.out.print(" Choose: ");
 switch (scanner.nextLine().trim()) {
 case "1" -> browseAndSelectMovie();
 case "2" -> viewMyTickets();
 case "3" -> topUpWallet();
 case "4" -> viewProfile();
 case "5" -> logout();
 default -> System.out.println(" [!] Invalid option.");
 }
 }
 private static void showAdminMenu() {
 System.out.println("\n" + LINE);
 System.out.printf(" ADMIN MENU · %s%n", currentUser.getUsername());
 System.out.println(LINE);
 System.out.println(" 1. Manage Movies");
 System.out.println(" 2. View All Users");
 System.out.println(" 3. Profile");
 System.out.println(" 4. Logout");
 System.out.println(LINE);
 System.out.print(" Choose: ");
 switch (scanner.nextLine().trim()) {
 case "1" -> manageMovies();
 case "2" -> userService.listAllUsers();
 case "3" -> viewProfile();
 case "4" -> logout();
 default -> System.out.println(" [!] Invalid option.");
 }
 }
 // ── Auth ────────────────────────────────────────────────
 private static void loginUser() {
 System.out.println("\n" + THIN);
 System.out.println(" LOGIN");
 System.out.println(THIN);
 System.out.print(" Username : ");
 String username = scanner.nextLine().trim();
 System.out.print(" Password : ");
 String password = scanner.nextLine();
 Optional<User> auth = userService.authenticateUser(username, password);
 if (auth.isPresent()) {
 currentUser = auth.get();
 System.out.printf("%n Welcome back, %s! (Role: %s)%n",
 currentUser.getUsername(), currentUser.getRole());
 } else {
 System.out.println(" [!] Invalid username or password.");
 }
 }
 private static void registerNewUser() {
 System.out.println("\n" + THIN);
 System.out.println(" REGISTER");
 System.out.println(THIN);
 // Username
 String username;
 while (true) {
 System.out.print(" Username : ");
 username = scanner.nextLine().trim();
 if (username.isEmpty()) { System.out.println(" [!] Username cannot be empty.");
continue; }
 if (userService.getUserByUsername(username).isPresent()) {
 System.out.println(" [!] Username taken. Try another.");
 } else { break; }
 }
 // Password
 String password;
 while (true) {
 System.out.print(" Password : ");
 password = scanner.nextLine();
 String err = Validator.validatePassword(password);
 if (err != null) { System.out.println(" [!] " + err); }
 else { break; }
 }
 // Email
 String email;
 while (true) {
 System.out.print(" Email : ");
 email = scanner.nextLine().trim();
 if (!Validator.isValidEmail(email)) {
 System.out.println(" [!] Invalid email. Use a proper format e.g. user@gmail.com");
 } else { break; }
 }
 userService.registerUser(username, password, email, User.UserRole.USER);
 }
 private static void logout() {
 System.out.println(" Logging out " + currentUser.getUsername() + "...");
 currentUser = null;
 }
 // ── Movie browsing & purchasing ─────────────────────────
 private static void browseAndSelectMovie() {
 List<Movie> movies = movieService.getAllMovies();
 if (movies.isEmpty()) { System.out.println(" No movies available."); return; }
 while (true) {
 System.out.println("\n" + THIN);
 System.out.println(" NOW SHOWING");
 System.out.println(THIN);
 for (int i = 0; i < movies.size(); i++) {
 Movie m = movies.get(i);
 System.out.printf(" [%d] %-36s %3d min ₱%.2f%n",
 i + 1, m.getTitle(), m.getDuration(), m.getPrice());
 System.out.printf(" Genre: %-30s Year: %d%n", m.getGenre(), m.getYear());
 }
 System.out.println(THIN);
 System.out.println(" [0] Back");
 System.out.print(" Select a movie: ");
 String input = scanner.nextLine().trim();
 if (input.equals("0")) return;
 int idx;
 try { idx = Integer.parseInt(input) - 1; }
 catch (NumberFormatException e) { System.out.println(" [!] Enter a number.");
continue; }
 if (idx < 0 || idx >= movies.size()) { System.out.println(" [!] Invalid selection.");
continue; }
 showMovieMenu(movies.get(idx));
 }
 }
 private static void showMovieMenu(Movie movie) {
 System.out.println("\n" + THIN);
 System.out.printf(" %s (%d)%n", movie.getTitle(), movie.getYear());
 System.out.println(THIN);
 System.out.printf(" Genre : %s%n", movie.getGenre());
 System.out.printf(" Duration : %d minutes%n", movie.getDuration());
 System.out.printf(" Description: %s%n", wordWrap(movie.getDescription(), 44, "
"));
 System.out.printf(" Ticket : ₱%.2f%n", movie.getPrice());
 System.out.println(THIN);
 System.out.println(" 1. Buy Ticket");
 System.out.println(" 2. Back");
 System.out.print(" Choose: ");
 switch (scanner.nextLine().trim()) {
 case "1" -> purchaseTicket(movie);
 case "2" -> { /* back */ }
 default -> System.out.println(" [!] Invalid option.");
 }
 }
 private static void purchaseTicket(Movie movie) {
 double ticketPrice = movie.getPrice();
 double serviceFee = Math.round(ticketPrice * 0.05 * 100.0) / 100.0;
 double total = ticketPrice + serviceFee;
 System.out.println("\n" + THIN);
 System.out.println(" CHECKOUT SUMMARY");
 System.out.println(THIN);
 System.out.printf(" Movie : %s%n", movie.getTitle());
 System.out.printf(" Ticket price: ₱%.2f%n", ticketPrice);
 System.out.printf(" Service fee : ₱%.2f (5%%)%n", serviceFee);
 System.out.printf(" %-13s ₱%.2f%n", "TOTAL:", total);
 System.out.println(THIN);
 System.out.printf(" Your balance: ₱%.2f%n", currentUser.getBalance());
 if (currentUser.getBalance() < total) {
 System.out.printf(" [!] Insufficient balance. You need ₱%.2f more.%n",
 total - currentUser.getBalance());
 System.out.println(" Go to 'Top Up Wallet' from the main menu.");
 return;
 }
 System.out.printf(" After payment: ₱%.2f%n", currentUser.getBalance() - total);
 System.out.println(THIN);
 System.out.print(" Confirm payment? (y/n): ");
 if (!scanner.nextLine().trim().equalsIgnoreCase("y")) {
 System.out.println(" Payment cancelled.");
 return;
 }
 currentUser.deductBalance(total);
 String bookingCode = generateBookingCode();
 Ticket ticket = new Ticket(bookingCode, movie, total, new Date());
 currentUser.addTicket(ticket);
 printTicket(ticket);
 }
 // ── Wallet ──────────────────────────────────────────────
 private static void topUpWallet() {
 System.out.println("\n" + THIN);
 System.out.println(" TOP UP WALLET");
 System.out.println(THIN);
 System.out.printf(" Current balance: ₱%.2f%n", currentUser.getBalance());
 System.out.println(THIN);
 System.out.println(" Enter the amount to add (numbers only).");
 System.out.println(" Example: 500 or 1000");
 System.out.print(" Amount: ₱");
 String raw = scanner.nextLine().trim();
 // Accept only digit characters (no decimals, no letters)
 if (!raw.matches("\\d+")) {
 System.out.println(" [!] Invalid amount. Enter whole numbers only (e.g. 500).");
 return;
 }
 double amount = Double.parseDouble(raw);
 if (amount <= 0) { System.out.println(" [!] Amount must be greater than zero."); return; }
 currentUser.addBalance(amount);
 System.out.printf("%n ₱%.2f added successfully!%n", amount);
 System.out.printf(" New balance: ₱%.2f%n", currentUser.getBalance());
 }
 // ── Tickets ─────────────────────────────────────────────
 private static void viewMyTickets() {
 List<Ticket> tickets = currentUser.getTickets();
 System.out.println("\n" + THIN);
 System.out.println(" MY TICKETS");
 System.out.println(THIN);
 if (tickets.isEmpty()) {
 System.out.println(" No tickets yet. Browse movies to buy one!");
 return;
 }
 for (int i = 0; i < tickets.size(); i++) {
 Ticket t = tickets.get(i);
 System.out.printf(" [%d] %s — ₱%.2f — Code: %s%n",
 i + 1, t.getMovie().getTitle(), t.getTotalPaid(), t.getBookingCode());
 }
 System.out.println(THIN);
 System.out.print(" View ticket QR? Enter number (or 0 to back): ");
 String input = scanner.nextLine().trim();
 if (input.equals("0")) return;
 try {
 int idx = Integer.parseInt(input) - 1;
 if (idx >= 0 && idx < tickets.size()) printTicket(tickets.get(idx));
 else System.out.println(" [!] Invalid selection.");
 } catch (NumberFormatException e) { System.out.println(" [!] Enter a number."); }
 }
 private static void viewProfile() {
 System.out.println("\n" + THIN);
 System.out.println(" PROFILE");
 System.out.println(THIN);
 System.out.printf(" Username : %s%n", currentUser.getUsername());
 System.out.printf(" Email : %s%n", currentUser.getEmail());
 System.out.printf(" Role : %s%n", currentUser.getRole());
 System.out.printf(" Balance : ₱%.2f%n", currentUser.getBalance());
 System.out.printf(" Tickets : %d purchased%n", currentUser.getTickets().size());
 System.out.println(THIN);
 }
 // ── Admin: Movie management ─────────────────────────────
 private static void manageMovies() {
 while (true) {
 System.out.println("\n" + THIN);
 System.out.println(" MANAGE MOVIES");
 System.out.println(THIN);
 System.out.println(" 1. List All Movies");
 System.out.println(" 2. Add New Movie");
 System.out.println(" 3. Delete Movie");
 System.out.println(" 4. Back");
 System.out.println(THIN);
 System.out.print(" Choose: ");
 switch (scanner.nextLine().trim()) {
 case "1" -> listAllMovies();
 case "2" -> addNewMovie();
 case "3" -> deleteMovie();
 case "4" -> { return; }
 default -> System.out.println(" [!] Invalid option.");
 }
 }
 }
 private static void listAllMovies() {
 List<Movie> movies = movieService.getAllMovies();
 System.out.println("\n" + THIN);
 System.out.println(" ALL MOVIES");
 System.out.println(THIN);
 if (movies.isEmpty()) { System.out.println(" No movies."); return; }
 for (int i = 0; i < movies.size(); i++) {
 Movie m = movies.get(i);
 System.out.printf(" [%d] %s (%d) — %d min — ₱%.2f%n",
 i + 1, m.getTitle(), m.getYear(), m.getDuration(), m.getPrice());
 }
 }
 private static void addNewMovie() {
 System.out.println("\n ADD MOVIE");
 System.out.println(THIN);
 System.out.print(" Title : "); String title = scanner.nextLine().trim();
 System.out.print(" Genre : "); String genre = scanner.nextLine().trim();
 System.out.print(" Description : "); String desc = scanner.nextLine().trim();
 System.out.print(" Duration (min): ");
 int duration;
 try { duration = Integer.parseInt(scanner.nextLine().trim()); }
 catch (NumberFormatException e) { System.out.println(" [!] Invalid duration."); return; }
 System.out.print(" Year : ");
 int year;
 try { year = Integer.parseInt(scanner.nextLine().trim()); }
 catch (NumberFormatException e) { System.out.println(" [!] Invalid year."); return; }
 System.out.print(" Price (₱) : ");
 double price;
 try { price = Double.parseDouble(scanner.nextLine().trim()); }
 catch (NumberFormatException e) { System.out.println(" [!] Invalid price."); return; }
 if (movieService.addMovie(title, genre, desc, duration, year, price)) {
 System.out.println(" Movie '" + title + "' added.");
 }
 }
 private static void deleteMovie() {
 listAllMovies();
 System.out.print("\n Enter title to delete: ");
 String title = scanner.nextLine().trim();
 if (movieService.deleteMovie(title)) System.out.println(" Deleted '" + title + "'.");
 else System.out.println(" [!] Movie not found.");
 }
 // ── Ticket printing & QR ────────────────────────────────
 private static void printTicket(Ticket ticket) {
 String code = ticket.getBookingCode();
 System.out.println("\n" + "╔" + "═".repeat(50) + "╗");
 System.out.println("║" + center(" CinéBook Official Ticket", 50) + "║");
 System.out.println("╠" + "═".repeat(50) + "╣");
 System.out.printf( "║ Movie : %-39s║%n", ticket.getMovie().getTitle());
 System.out.printf( "║ Genre : %-39s║%n", ticket.getMovie().getGenre());
 System.out.printf( "║ Year : %-39s║%n", ticket.getMovie().getYear());
 System.out.printf( "║ Holder : %-39s║%n", currentUser.getUsername());
 System.out.printf( "║ Paid : ₱%-38.2f║%n", ticket.getTotalPaid());
 System.out.printf( "║ Date : %-39s║%n", ticket.getPurchaseDate());
 System.out.println("╠" + "═".repeat(50) + "╣");
 System.out.println("║" + center("▐ BOOKING CODE: " + code + " ▌", 50) + "║");
 System.out.println("╠" + "═".repeat(50) + "╣");
 printAsciiQR(code);
 System.out.println("╚" + "═".repeat(50) + "╝");
 }
 /**
 * Prints a deterministic 13×13 ASCII "QR code" derived from the booking code.
 * Each character in the code seeds a simple hash to produce a unique pattern.
 * Finder patterns (top-left, top-right, bottom-left) match real QR convention.
 */
 private static void printAsciiQR(String code) {
 int SIZE = 13;
 boolean[][] grid = new boolean[SIZE][SIZE];
 // Seed from booking code
 int seed = 7;
 for (char c : code.toCharArray()) seed = seed * 31 + c;
 // Fill interior with pseudo-random data
 Random rng = new Random(seed);
 for (int r = 0; r < SIZE; r++)
 for (int c = 0; c < SIZE; c++)
 grid[r][c] = rng.nextBoolean();
 // Draw three finder patterns (7×7 in real QR → scaled to 3×3 here)
 drawFinder(grid, 0, 0);
 drawFinder(grid, 0, SIZE - 3);
 drawFinder(grid, SIZE - 3, 0);
 for (int r = 0; r < SIZE; r++) {
 System.out.print("║ ");
 for (int c = 0; c < SIZE; c++)
 System.out.print(grid[r][c] ? "██" : " ");
 System.out.println(" ║");
 }
 }
 private static void drawFinder(boolean[][] grid, int row, int col) {
 for (int r = 0; r < 3; r++)
 for (int c = 0; c < 3; c++)
 grid[row + r][col + c] = (r == 0 || r == 2 || c == 0 || c == 2) || (r == 1 && c == 1);
 }
 // ── Utilities ───────────────────────────────────────────
 private static String generateBookingCode() {
 String chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
 StringBuilder sb = new StringBuilder("CB-");
 Random rng = new Random();
 for (int i = 0; i < 6; i++) sb.append(chars.charAt(rng.nextInt(chars.length())));
 return sb.toString();
 }
 private static String center(String text, int width) {
 // Strip emoji for length calculation (rough)
 int len = text.codePointCount(0, text.length());
 int pad = Math.max(0, width - len);
 int left = pad / 2, right = pad - left;
 return " ".repeat(left) + text + " ".repeat(right);
 }
 private static String wordWrap(String text, int width, String indent) {
 if (text.length() <= width) return text;
 int cut = text.lastIndexOf(' ', width);
 if (cut == -1) cut = width;
 return text.substring(0, cut) + "\n" + indent + wordWrap(text.substring(cut + 1), width,
indent);
 }
 private static void printBanner() {
 System.out.println("\n" + LINE);
 System.out.println(" Your cinema booking system. Buy tickets, watch great films.");
 System.out.println(LINE);
 }
 //
═══════════════════════════════════════════════════════
═
 // VALIDATOR
 //
═══════════════════════════════════════════════════════
═
 static class Validator {
 // RFC-5322 simplified — accepts standard email formats (gmail.com, yahoo.com,
etc.)
 private static final Pattern EMAIL_PATTERN = Pattern.compile(
 "^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"
 );
 /**
 * Returns an error string if invalid, or null if password is acceptable.
 * Rules: 8+ chars, at least 1 uppercase letter, at least 1 digit.
 */
 static String validatePassword(String password) {
 if (password == null || password.length() < 8)
 return "Password must be at least 8 characters long.";
 if (!password.chars().anyMatch(Character::isUpperCase))
 return "Password must contain at least 1 uppercase letter (A-Z).";
 if (!password.chars().anyMatch(Character::isDigit))
 return "Password must contain at least 1 number (0-9).";
 return null;
 }
 static boolean isValidEmail(String email) {
 return email != null && EMAIL_PATTERN.matcher(email).matches();
 }
 }
 //
═══════════════════════════════════════════════════════
═
 // USER MODEL
 //
═══════════════════════════════════════════════════════
═
 static class User {
 public enum UserRole { USER, ADMIN }
 private final String username;
 private final String hashedPassword;
 private final String email;
 private final UserRole role;
 private double balance;
 private final List<Ticket> tickets = new ArrayList<>();
 public User(String username, String password, String email, UserRole role, double
initialBalance) {
 this.username = username;
 this.email = email;
 this.role = role;
 this.balance = initialBalance;
 this.hashedPassword = hashPassword(password);
 }
 public boolean verifyPassword(String plain) {
 return this.hashedPassword != null &&
this.hashedPassword.equals(hashPassword(plain));
 }
 private String hashPassword(String plain) {
 try {
 MessageDigest md = MessageDigest.getInstance("SHA-256");
 byte[] hash = md.digest(plain.getBytes());
 return Base64.getEncoder().encodeToString(hash);
 } catch (NoSuchAlgorithmException e) {
 System.err.println(" [!] SHA-256 not available: " + e.getMessage());
 return null;
 }
 }
 public void addBalance(double amount) { balance += amount; }
 public void deductBalance(double amount) { balance = Math.round((balance - amount)
* 100.0) / 100.0; }
 public void addTicket(Ticket t) { tickets.add(t); }
 public String getUsername() { return username; }
 public String getEmail() { return email; }
 public UserRole getRole() { return role; }
 public double getBalance() { return balance; }
 public List<Ticket> getTickets() { return Collections.unmodifiableList(tickets); }
 @Override
 public String toString() {
 return String.format(" %-15s %-28s %-7s ₱%.2f tickets=%d",
 username, email, role, balance, tickets.size());
 }
 }
 //
═══════════════════════════════════════════════════════
═
 // USER SERVICE
 //
═══════════════════════════════════════════════════════
═
 static class UserService {
 private final Map<String, User> users = new LinkedHashMap<>();
 public UserService() {
 // Pre-seeded accounts — passwords meet the new validation rules
 registerUser("admin", "Admin1234", "admin@cinebook.com",
User.UserRole.ADMIN, 0);
 registerUser("john.doe", "JohnPass1", "john@gmail.com", User.UserRole.USER,
500);
 registerUser("maria", "Maria2025", "maria@yahoo.com", User.UserRole.USER,
300);
 }
 public Optional<User> registerUser(String username, String password,
 String email, User.UserRole role) {
 return registerUser(username, password, email, role, 0);
 }
 public Optional<User> registerUser(String username, String password,
 String email, User.UserRole role, double initialBalance) {
 if (users.containsKey(username)) {
 System.out.println(" [!] Username '" + username + "' already exists.");
 return Optional.empty();
 }
 User u = new User(username, password, email, role, initialBalance);
 users.put(username, u);
 System.out.println(" Registered '" + username + "' as " + role + ".");
 return Optional.of(u);
 }
 public Optional<User> authenticateUser(String username, String password) {
 User u = users.get(username);
 if (u != null && u.verifyPassword(password)) return Optional.of(u);
 return Optional.empty();
 }
 public Optional<User> getUserByUsername(String username) {
 return Optional.ofNullable(users.get(username));
 }
 public void listAllUsers() {
 System.out.println("\n" + THIN);
 System.out.println(" ALL USERS");
 System.out.printf(" %-15s %-28s %-7s %s%n", "Username", "Email", "Role",
"Balance");
 System.out.println(THIN);
 users.values().forEach(System.out::println);
 System.out.println(THIN);
 }
 }
 //
═══════════════════════════════════════════════════════
═
 // MOVIE MODEL
 //
═══════════════════════════════════════════════════════
═
 static class Movie {
 private final String title;
 private String genre;
 private String description;
 private int duration; // minutes
 private int year;
 private double price; // PHP
 public Movie(String title, String genre, String description,
 int duration, int year, double price) {
 this.title = title;
 this.genre = genre;
 this.description = description;
 this.duration = duration;
 this.year = year;
 this.price = price;
 }
 public String getTitle() { return title; }
 public String getGenre() { return genre; }
 public String getDescription() { return description; }
 public int getDuration() { return duration; }
 public int getYear() { return year; }
 public double getPrice() { return price; }
 public void setGenre(String genre) { this.genre = genre; }
 public void setDescription(String description) { this.description = description; }
 public void setDuration(int duration) { this.duration = duration; }
 public void setPrice(double price) { this.price = price; }
 @Override
 public String toString() {
 return String.format("%s (%d) — %s — %d min — ₱%.2f", title, year, genre, duration,
price);
 }
 }
 //
═══════════════════════════════════════════════════════
═
 // MOVIE SERVICE
 //
═══════════════════════════════════════════════════════
═
 static class MovieService {
 private final Map<String, Movie> movies = new LinkedHashMap<>();
 public MovieService() {
 // Real movies with Philippine cinema prices (₱)
 addMovie("Oppenheimer",
 "Biography / Drama",
 "The story of J. Robert Oppenheimer and his pivotal role in developing the atomic
bomb during World War II.",
 180, 2023, 380.00);
 addMovie("Dune: Part Two",
 "Sci-Fi / Adventure",
 "Paul Atreides unites with the Fremen of Arrakis as he seeks revenge against
those who destroyed his family and their way of life.",
 166, 2024, 420.00);
 addMovie("The Dark Knight",
 "Action / Crime",
 "Batman faces the Joker, a criminal mastermind who plunges Gotham City into
anarchy with his reign of chaos.",
 152, 2008, 300.00);
 addMovie("Interstellar",
 "Sci-Fi / Drama",
 "A team of explorers travel through a wormhole in space in an attempt to ensure
humanity's survival on a dying Earth.",
 169, 2014, 320.00);
 addMovie("Spider-Man: No Way Home",
 "Action / Superhero",
 "With his identity revealed, Peter Parker asks Doctor Strange for help —
unleashing a dangerous multiverse of villains.",
 148, 2021, 350.00);
 addMovie("Everything Everywhere All at Once",
 "Comedy / Sci-Fi",
 "An aging Chinese immigrant discovers she alone can save existence by exploring
alternate universes and connecting with her family.",
 139, 2022, 280.00);
 addMovie("Avengers: Endgame",
 "Action / Superhero",
 "The surviving Avengers assemble one final time to reverse Thanos's devastating
snap and restore the universe.",
 181, 2019, 370.00);
 addMovie("Parasite",
 "Thriller / Drama",
 "Greed and class discrimination threaten the newly formed symbiotic
relationship between the wealthy Park family and the destitute Kim clan.",
 132, 2019, 260.00);
 }
 /**
 * Add a movie. Returns false if a movie with the same title already exists.
 */
 public boolean addMovie(String title, String genre, String description,
 int duration, int year, double price) {
 if (movies.containsKey(title)) {
 System.out.println(" [!] Movie '" + title + "' already exists.");
 return false;
 }
 movies.put(title, new Movie(title, genre, description, duration, year, price));
 return true;
 }
 public boolean deleteMovie(String title) {
 return movies.remove(title) != null;
 }
 public Optional<Movie> getMovieByTitle(String title) {
 return Optional.ofNullable(movies.get(title));
 }
 public List<Movie> getAllMovies() {
 return new ArrayList<>(movies.values());
 }
 }
 //
═══════════════════════════════════════════════════════
═
 // TICKET MODEL
 //
═══════════════════════════════════════════════════════
═
 static class Ticket {
 private final String bookingCode;
 private final Movie movie;
 private final double totalPaid;
 private final Date purchaseDate;
 public Ticket(String bookingCode, Movie movie, double totalPaid, Date purchaseDate) {
 this.bookingCode = bookingCode;
 this.movie = movie;
 this.totalPaid = totalPaid;
 this.purchaseDate = purchaseDate;
 }
 public String getBookingCode() { return bookingCode; }
 public Movie getMovie() { return movie; }
 public double getTotalPaid() { return totalPaid; }
 public Date getPurchaseDate() { return purchaseDate; }
 }
}
