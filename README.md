
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <fstream>
#include <iomanip>
#include <memory>
#include <map>

class Product {
private:
    int id;
    std::string name;
    double price;
    int quantity;
    std::string category;

public:
    Product(int id, const std::string& name, double price, int quantity, const std::string& category)
        : id(id), name(name), price(price), quantity(quantity), category(category) {}

    // Getters
    int getId() const { return id; }
    std::string getName() const { return name; }
    double getPrice() const { return price; }
    int getQuantity() const { return quantity; }
    std::string getCategory() const { return category; }

    // Setters
    void setPrice(double newPrice) { price = newPrice; }
    void setQuantity(int newQuantity) { quantity = newQuantity; }

    // Methods
    void addStock(int amount) { quantity += amount; }
    bool removeStock(int amount) {
        if (quantity >= amount) {
            quantity -= amount;
            return true;
        }
        return false;
    }

    void display() const {
        std::cout << std::setw(5) << id << " | "
                  << std::setw(20) << name << " | "
                  << std::setw(10) << std::fixed << std::setprecision(2) << price << " | "
                  << std::setw(8) << quantity << " | "
                  << std::setw(15) << category << std::endl;
    }
};

class User {
private:
    int id;
    std::string username;
    std::string password;
    std::string role; // "admin" sau "client"

public:
    User(int id, const std::string& username, const std::string& password, const std::string& role)
        : id(id), username(username), password(password), role(role) {}

    // Getters
    int getId() const { return id; }
    std::string getUsername() const { return username; }
    std::string getRole() const { return role; }

    // Authentication
    bool authenticate(const std::string& inputPassword) const {
        return password == inputPassword;
    }

    void display() const {
        std::cout << "ID: " << id << " | Username: " << username
                  << " | Role: " << role << std::endl;
    }
};

class Order {
private:
    int id;
    int userId;
    std::vector<std::pair<int, int>> items; // pair<product_id, quantity>
    std::string status; // "pending", "shipped", "delivered"
    double totalAmount;

public:
    Order(int id, int userId)
        : id(id), userId(userId), status("pending"), totalAmount(0.0) {}

    // Getters
    int getId() const { return id; }
    int getUserId() const { return userId; }
    std::string getStatus() const { return status; }
    double getTotalAmount() const { return totalAmount; }
    const std::vector<std::pair<int, int>>& getItems() const { return items; }

    // Setters
    void setStatus(const std::string& newStatus) { status = newStatus; }
    void setTotalAmount(double amount) { totalAmount = amount; }

    // Methods
    void addItem(int productId, int quantity) {
        // Check if product already exists in order
        auto it = std::find_if(items.begin(), items.end(),
                               [productId](const std::pair<int, int>& item) {
                                   return item.first == productId;
                               });

        if (it != items.end()) {
            // Update quantity if product already exists
            it->second += quantity;
        } else {
            // Add new product to order
            items.push_back(std::make_pair(productId, quantity));
        }
    }

    void display() const {
        std::cout << "Order ID: " << id << " | User ID: " << userId
                  << " | Status: " << status
                  << " | Total: " << std::fixed << std::setprecision(2) << totalAmount << std::endl;
    }
};

class StoreManager {
private:
    std::vector<std::shared_ptr<Product>> products;
    std::vector<std::shared_ptr<User>> users;
    std::vector<std::shared_ptr<Order>> orders;
    int nextProductId = 1;
    int nextUserId = 1;
    int nextOrderId = 1;
    std::shared_ptr<User> currentUser = nullptr;

    // File paths
    const std::string PRODUCTS_FILE = "products.txt";
    const std::string USERS_FILE = "users.txt";
    const std::string ORDERS_FILE = "orders.txt";

public:
    StoreManager() {
        loadData();
    }

    ~StoreManager() {
        saveData();
    }

    // User Management
    bool registerUser(const std::string& username, const std::string& password, const std::string& role) {
        // Check if username already exists
        auto it = std::find_if(users.begin(), users.end(),
                              [&username](const std::shared_ptr<User>& user) {
                                  return user->getUsername() == username;
                              });

        if (it != users.end()) {
            std::cout << "Username already exists!" << std::endl;
            return false;
        }

        // Create new user
        users.push_back(std::make_shared<User>(nextUserId++, username, password, role));
        std::cout << "User registered successfully!" << std::endl;
        return true;
    }

    bool login(const std::string& username, const std::string& password) {
        auto it = std::find_if(users.begin(), users.end(),
                              [&username](const std::shared_ptr<User>& user) {
                                  return user->getUsername() == username;
                              });

        if (it != users.end() && (*it)->authenticate(password)) {
            currentUser = *it;
            std::cout << "Login successful! Welcome, " << username << "!" << std::endl;
            return true;
        }

        std::cout << "Invalid username or password!" << std::endl;
        return false;
    }

    void logout() {
        if (currentUser) {
            std::cout << "Logout successful! Goodbye, " << currentUser->getUsername() << "!" << std::endl;
            currentUser = nullptr;
        } else {
            std::cout << "No user is currently logged in!" << std::endl;
        }
    }

    // Product Management
    void addProduct(const std::string& name, double price, int quantity, const std::string& category) {
        if (!isAdmin()) {
            std::cout << "Permission denied! Only admins can add products." << std::endl;
            return;
        }

        products.push_back(std::make_shared<Product>(
            nextProductId++, name, price, quantity, category
        ));
        std::cout << "Product added successfully!" << std::endl;
    }

    void updateProduct(int productId, double newPrice, int newQuantity) {
        if (!isAdmin()) {
            std::cout << "Permission denied! Only admins can update products." << std::endl;
            return;
        }

        auto it = findProduct(productId);
        if (it != products.end()) {
            (*it)->setPrice(newPrice);
            (*it)->setQuantity(newQuantity);
            std::cout << "Product updated successfully!" << std::endl;
        } else {
            std::cout << "Product not found!" << std::endl;
        }
    }

    void deleteProduct(int productId) {
        if (!isAdmin()) {
            std::cout << "Permission denied! Only admins can delete products." << std::endl;
            return;
        }

        auto it = findProduct(productId);
        if (it != products.end()) {
            products.erase(it);
            std::cout << "Product deleted successfully!" << std::endl;
        } else {
            std::cout << "Product not found!" << std::endl;
        }
    }

    // Order Management
    void createOrder() {
        if (!isLoggedIn()) {
            std::cout << "Please login first!" << std::endl;
            return;
        }

        auto order = std::make_shared<Order>(nextOrderId++, currentUser->getId());
        orders.push_back(order);
        std::cout << "Order created with ID: " << order->getId() << std::endl;
    }

    void addToOrder(int orderId, int productId, int quantity) {
        if (!isLoggedIn()) {
            std::cout << "Please login first!" << std::endl;
            return;
        }

        auto orderIt = findOrder(orderId);
        if (orderIt == orders.end()) {
            std::cout << "Order not found!" << std::endl;
            return;
        }

        auto productIt = findProduct(productId);
        if (productIt == products.end()) {
            std::cout << "Product not found!" << std::endl;
            return;
        }

        if ((*orderIt)->getUserId() != currentUser->getId() && !isAdmin()) {
            std::cout << "Permission denied! You can only modify your own orders." << std::endl;
            return;
        }

        if ((*productIt)->getQuantity() < quantity) {
            std::cout << "Not enough stock available!" << std::endl;
            return;
        }

        (*orderIt)->addItem(productId, quantity);
        (*productIt)->removeStock(quantity);

        // Update total amount
        double total = (*orderIt)->getTotalAmount();
        total += (*productIt)->getPrice() * quantity;
        (*orderIt)->setTotalAmount(total);

        std::cout << "Item added to order!" << std::endl;
    }

    void processOrder(int orderId, const std::string& status) {
        if (!isAdmin()) {
            std::cout << "Permission denied! Only admins can process orders." << std::endl;
            return;
        }

        auto orderIt = findOrder(orderId);
        if (orderIt == orders.end()) {
            std::cout << "Order not found!" << std::endl;
            return;
        }

        (*orderIt)->setStatus(status);
        std::cout << "Order status updated to: " << status << std::endl;
    }

    // Display Methods
    void displayProducts(const std::string& category = "") {
        std::cout << "\n====== PRODUCTS ======" << std::endl;
        std::cout << std::setw(5) << "ID" << " | "
                  << std::setw(20) << "Name" << " | "
                  << std::setw(10) << "Price" << " | "
                  << std::setw(8) << "Quantity" << " | "
                  << std::setw(15) << "Category" << std::endl;
        std::cout << std::string(65, '-') << std::endl;

        for (const auto& product : products) {
            if (category.empty() || product->getCategory() == category) {
                product->display();
            }
        }
        std::cout << std::endl;
    }

    void displayUsers() {
        if (!isAdmin()) {
            std::cout << "Permission denied! Only admins can view users." << std::endl;
            return;
        }

        std::cout << "\n====== USERS ======" << std::endl;
        for (const auto& user : users) {
            user->display();
        }
        std::cout << std::endl;
    }

    void displayOrders(bool onlyMine = false) {
        if (!isLoggedIn()) {
            std::cout << "Please login first!" << std::endl;
            return;
        }

        std::cout << "\n====== ORDERS ======" << std::endl;
        for (const auto& order : orders) {
            if (!onlyMine || order->getUserId() == currentUser->getId() || isAdmin()) {
                order->display();

                std::cout << "Items:" << std::endl;
                for (const auto& item : order->getItems()) {
                    auto productIt = findProduct(item.first);
                    if (productIt != products.end()) {
                        std::cout << "  - " << (*productIt)->getName()
                                  << " x " << item.second
                                  << " = " << (*productIt)->getPrice() * item.second << std::endl;
                    }
                }
                std::cout << std::endl;
            }
        }
    }

    // Statistics and Reports
    void generateSalesReport() {
        if (!isAdmin()) {
            std::cout << "Permission denied! Only admins can generate reports." << std::endl;
            return;
        }

        std::map<std::string, double> categoryRevenue;
        std::map<int, double> productRevenue;
        double totalRevenue = 0.0;

        for (const auto& order : orders) {
            if (order->getStatus() == "delivered") {
                for (const auto& item : order->getItems()) {
                    auto productIt = findProduct(item.first);
                    if (productIt != products.end()) {
                        double itemRevenue = (*productIt)->getPrice() * item.second;
                        totalRevenue += itemRevenue;
                        productRevenue[item.first] += itemRevenue;
                        categoryRevenue[(*productIt)->getCategory()] += itemRevenue;
                    }
                }
            }
        }

        std::cout << "\n====== SALES REPORT ======" << std::endl;
        std::cout << "Total Revenue: $" << std::fixed << std::setprecision(2) << totalRevenue << std::endl;

        std::cout << "\nRevenue by Category:" << std::endl;
        for (const auto& pair : categoryRevenue) {
            std::cout << std::setw(15) << pair.first << ": $"
                      << std::fixed << std::setprecision(2) << pair.second << std::endl;
        }

        std::cout << "\nTop Products by Revenue:" << std::endl;
        std::vector<std::pair<int, double>> sortedProducts(productRevenue.begin(), productRevenue.end());
        std::sort(sortedProducts.begin(), sortedProducts.end(),
                 [](const std::pair<int, double>& a, const std::pair<int, double>& b) {
                     return a.second > b.second;
                 });

        int count = 0;
        for (const auto& pair : sortedProducts) {
            if (count++ >= 5) break; // Top 5 products

            auto productIt = findProduct(pair.first);
            if (productIt != products.end()) {
                std::cout << std::setw(20) << (*productIt)->getName() << ": $"
                          << std::fixed << std::setprecision(2) << pair.second << std::endl;
            }
        }
        std::cout << std::endl;
    }

    // File Operations
    void loadData() {
        loadProducts();
        loadUsers();
        loadOrders();
    }

    void saveData() {
        saveProducts();
        saveUsers();
        saveOrders();
    }

private:
    // Helper Methods
    bool isLoggedIn() const {
        return currentUser != nullptr;
    }

    bool isAdmin() const {
        return isLoggedIn() && currentUser->getRole() == "admin";
    }

    std::vector<std::shared_ptr<Product>>::iterator findProduct(int productId) {
        return std::find_if(products.begin(), products.end(),
                           [productId](const std::shared_ptr<Product>& product) {
                               return product->getId() == productId;
                           });
    }

    std::vector<std::shared_ptr<Order>>::iterator findOrder(int orderId) {
        return std::find_if(orders.begin(), orders.end(),
                           [orderId](const std::shared_ptr<Order>& order) {
                               return order->getId() == orderId;
                           });
    }

    // File Operations
    void loadProducts() {
        std::ifstream file(PRODUCTS_FILE);
        if (!file) return;

        products.clear();
        nextProductId = 1;

        int id, quantity;
        std::string name, category;
        double price;

        while (file >> id >> std::ws) {
            std::getline(file, name, '|');
            file >> price >> quantity >> std::ws;
            std::getline(file, category);

            products.push_back(std::make_shared<Product>(id, name, price, quantity, category));
            if (id >= nextProductId) nextProductId = id + 1;
        }

        file.close();
    }

    void saveProducts() {
        std::ofstream file(PRODUCTS_FILE);
        if (!file) {
            std::cerr << "Error opening file for writing: " << PRODUCTS_FILE << std::endl;
            return;
        }

        for (const auto& product : products) {
            file << product->getId() << " "
                 << product->getName() << "|"
                 << product->getPrice() << " "
                 << product->getQuantity() << " "
                 << product->getCategory() << std::endl;
        }

        file.close();
    }

    void loadUsers() {
        std::ifstream file(USERS_FILE);
        if (!file) return;

        users.clear();
        nextUserId = 1;

        int id;
        std::string username, password, role;

        while (file >> id >> username >> password >> role) {
            users.push_back(std::make_shared<User>(id, username, password, role));
            if (id >= nextUserId) nextUserId = id + 1;
        }

        file.close();
    }

    void saveUsers() {
        std::ofstream file(USERS_FILE);
        if (!file) {
            std::cerr << "Error opening file for writing: " << USERS_FILE << std::endl;
            return;
        }

        for (const auto& user : users) {
            file << user->getId() << " "
                 << user->getUsername() << " "
                 << "password" << " " // In a real system, this would be encrypted
                 << user->getRole() << std::endl;
        }

        file.close();
    }

    void loadOrders() {
        std::ifstream file(ORDERS_FILE);
        if (!file) return;

        orders.clear();
        nextOrderId = 1;

        int id, userId, productId, quantity, itemCount;
        double totalAmount;
        std::string status;

        while (file >> id >> userId >> status >> totalAmount >> itemCount) {
            auto order = std::make_shared<Order>(id, userId);
            order->setStatus(status);
            order->setTotalAmount(totalAmount);

            for (int i = 0; i < itemCount; i++) {
                file >> productId >> quantity;
                order->addItem(productId, quantity);
            }

            orders.push_back(order);
            if (id >= nextOrderId) nextOrderId = id + 1;
        }

        file.close();
    }

    void saveOrders() {
        std::ofstream file(ORDERS_FILE);
        if (!file) {
            std::cerr << "Error opening file for writing: " << ORDERS_FILE << std::endl;
            return;
        }

        for (const auto& order : orders) {
            file << order->getId() << " "
                 << order->getUserId() << " "
                 << order->getStatus() << " "
                 << order->getTotalAmount() << " "
                 << order->getItems().size() << " ";

            for (const auto& item : order->getItems()) {
                file << item.first << " " << item.second << " ";
            }
            file << std::endl;
        }

        file.close();
    }
};

int main() {
    StoreManager store;
    bool running = true;
    std::string command;

    // Create admin user if it doesn't exist
    store.registerUser("admin", "admin123", "admin");

    while (running) {
        std::cout << "\n===== STORE MANAGEMENT SYSTEM =====\n";
        std::cout << "1. Register User\n";
        std::cout << "2. Login\n";
        std::cout << "3. Logout\n";
        std::cout << "4. Display Products\n";
        std::cout << "5. Add Product\n";
        std::cout << "6. Update Product\n";
        std::cout << "7. Delete Product\n";
        std::cout << "8. Create Order\n";
        std::cout << "9. Add Item to Order\n";
        std::cout << "10. Process Order\n";
        std::cout << "11. Display Orders\n";
        std::cout << "12. Display Users\n";
        std::cout << "13. Generate Sales Report\n";
        std::cout << "14. Exit\n";
        std::cout << "Enter your choice: ";
        std::getline(std::cin, command);

        if (command == "1") {
            std::string username, password, role;
            std::cout << "Enter username: ";
            std::getline(std::cin, username);
            std::cout << "Enter password: ";
            std::getline(std::cin, password);
            std::cout << "Enter role (admin/client): ";
            std::getline(std::cin, role);
            store.registerUser(username, password, role);
        }
        else if (command == "2") {
            std::string username, password;
            std::cout << "Enter username: ";
            std::getline(std::cin, username);
            std::cout << "Enter password: ";
            std::getline(std::cin, password);
            store.login(username, password);
        }
        else if (command == "3") {
            store.logout();
        }
        else if (command == "4") {
            std::string category;
            std::cout << "Enter category (leave empty for all): ";
            std::getline(std::cin, category);
            store.displayProducts(category);
        }
        else if (command == "5") {
            std::string name, category;
            double price;
            int quantity;

            std::cout << "Enter product name: ";
            std::getline(std::cin, name);

            std::cout << "Enter price: ";
            std::cin >> price;
            std::cin.ignore();

            std::cout << "Enter quantity: ";
            std::cin >> quantity;
            std::cin.ignore();

            std::cout << "Enter category: ";
            std::getline(std::cin, category);

            store.addProduct(name, price, quantity, category);
        }
        else if (command == "6") {
            int id;
            double price;
            int quantity;

            std::cout << "Enter product ID: ";
            std::cin >> id;
            std::cin.ignore();

            std::cout << "Enter new price: ";
            std::cin >> price;
            std::cin.ignore();

            std::cout << "Enter new quantity: ";
            std::cin >> quantity;
            std::cin.ignore();

            store.updateProduct(id, price, quantity);
        }
        else if (command == "7") {
            int id;
            std::cout << "Enter product ID: ";
            std::cin >> id;
            std::cin.ignore();

            store.deleteProduct(id);
        }
        else if (command == "8") {
            store.createOrder();
        }
        else if (command == "9") {
            int orderId, productId, quantity;

            std::cout << "Enter order ID: ";
            std::cin >> orderId;
            std::cin.ignore();

            std::cout << "Enter product ID: ";
            std::cin >> productId;
            std::cin.ignore();

            std::cout << "Enter quantity: ";
            std::cin >> quantity;
            std::cin.ignore();

            store.addToOrder(orderId, productId, quantity);
        }
        else if (command == "10") {
            int orderId;
            std::string status;

            std::cout << "Enter order ID: ";
            std::cin >> orderId;
            std::cin.ignore();

            std::cout << "Enter new status (pending/shipped/delivered): ";
            std::getline(std::cin, status);

            store.processOrder(orderId, status);
        }
        else if (command == "11") {
            char choice;
            std::cout << "Display only my orders? (y/n): ";
            std::cin >> choice;
            std::cin.ignore();

            store.displayOrders(choice == 'y' || choice == 'Y');
        }
        else if (command == "12") {
            store.displayUsers();
        }
        else if (command == "13") {
            store.generateSalesReport();
        }
        else if (command == "14") {
            std::cout << "Thank you for using Store Management System. Goodbye!" << std::endl;
            running = false;
        }
        else {
            std::cout << "Invalid command. Please try again." << std::endl;
        }
    }

    return 0;
}
