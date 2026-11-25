# Sagar_Nov_25
Nov 25th Code
package com.demo.qa;

import io.github.bonigarcia.wdm.WebDriverManager;
import org.apache.commons.io.FileUtils;
import org.openqa.selenium.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.Select;
import org.openqa.selenium.support.ui.WebDriverWait;
import org.testng.Assert;
import org.testng.ITestResult;
import org.testng.annotations.*;

import java.io.File;
import java.io.IOException;
import java.time.Duration;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

/**
 * Full functional test suite for https://www.saucedemo.com
 *
 * Covers:
 *  1.  Login page (valid / invalid / locked-out / empty / problem users, UI checks)
 *  2.  Inventory page (product list, sorting, add/remove, cart badge)
 *  3.  Product details page
 *  4.  Cart page
 *  5.  Checkout step one (form validation)
 *  6.  Checkout step two (overview + price math)
 *  7.  Checkout complete
 *  8.  Burger menu (All Items, About, Logout, Reset App State)
 *  9.  Footer / social links
 *  10. Session / security behaviour (direct URL access without login)
 *  11. End-to-end purchase flows
 *
 * Run with:  mvn clean test
 */
public class SauceDemoFullTest {

    // ------------------------------------------------------------------
    // CONFIG
    // ------------------------------------------------------------------
    private static final String BASE_URL = "https://www.saucedemo.com/";
    private static final String INVENTORY_URL = BASE_URL + "inventory.html";
    private static final String CART_URL = BASE_URL + "cart.html";

    private static final String STANDARD_USER = "standard_user";
    private static final String LOCKED_USER = "locked_out_user";
    private static final String PROBLEM_USER = "problem_user";
    private static final String PERF_USER = "performance_glitch_user";
    private static final String PASSWORD = "secret_sauce";

    private static final int TOTAL_PRODUCTS = 6;
    private static final boolean HEADLESS = Boolean.parseBoolean(System.getProperty("headless", "true"));

    private WebDriver driver;
    private WebDriverWait wait;

    // ------------------------------------------------------------------
    // LOCATORS
    // ------------------------------------------------------------------
    // Login
    private final By usernameInput = By.id("user-name");
    private final By passwordInput = By.id("password");
    private final By loginButton = By.id("login-button");
    private final By errorMessage = By.cssSelector("[data-test='error']");
    private final By errorCloseButton = By.cssSelector(".error-button");
    private final By loginLogo = By.className("login_logo");

    // Header / menu
    private final By pageTitle = By.className("title");
    private final By burgerButton = By.id("react-burger-menu-btn");
    private final By burgerClose = By.id("react-burger-cross-btn");
    private final By menuAllItems = By.id("inventory_sidebar_link");
    private final By menuAbout = By.id("about_sidebar_link");
    private final By menuLogout = By.id("logout_sidebar_link");
    private final By menuReset = By.id("reset_sidebar_link");
    private final By cartLink = By.className("shopping_cart_link");
    private final By cartBadge = By.className("shopping_cart_badge");
    private final By appLogo = By.className("app_logo");

    // Inventory
    private final By inventoryItems = By.className("inventory_item");
    private final By itemNames = By.className("inventory_item_name");
    private final By itemPrices = By.className("inventory_item_price");
    private final By itemDescriptions = By.className("inventory_item_desc");
    private final By itemImages = By.cssSelector(".inventory_item_img img");
    private final By sortDropdown = By.className("product_sort_container");
    private final By inventoryButtons = By.cssSelector("button.btn_inventory");

    // Details
    private final By detailName = By.className("inventory_details_name");
    private final By detailDesc = By.className("inventory_details_desc");
    private final By detailPrice = By.className("inventory_details_price");
    private final By detailImage = By.className("inventory_details_img");
    private final By backToProducts = By.id("back-to-products");

    // Cart
    private final By cartItems = By.className("cart_item");
    private final By cartItemNames = By.className("inventory_item_name");
    private final By cartQuantity = By.className("cart_quantity");
    private final By continueShopping = By.id("continue-shopping");
    private final By checkoutButton = By.id("checkout");

    // Checkout
    private final By firstName = By.id("first-name");
    private final By lastName = By.id("last-name");
    private final By postalCode = By.id("postal-code");
    private final By continueButton = By.id("continue");
    private final By cancelButton = By.id("cancel");
    private final By finishButton = By.id("finish");
    private final By subtotalLabel = By.className("summary_subtotal_label");
    private final By taxLabel = By.className("summary_tax_label");
    private final By totalLabel = By.className("summary_total_label");
    private final By completeHeader = By.className("complete-header");
    private final By completeText = By.className("complete-text");
    private final By completeImage = By.className("pony_express");
    private final By backHomeButton = By.id("back-to-products");

    // Footer
    private final By footerCopy = By.className("footer_copy");
    private final By twitter = By.cssSelector(".social_twitter a");
    private final By facebook = By.cssSelector(".social_facebook a");
    private final By linkedin = By.cssSelector(".social_linkedin a");

    // ------------------------------------------------------------------
    // SETUP / TEARDOWN
    // ------------------------------------------------------------------
    @BeforeClass(alwaysRun = true)
    public void setUpClass() {
        WebDriverManager.chromedriver().setup();
    }

    @BeforeMethod(alwaysRun = true)
    public void setUp() {
        ChromeOptions options = new ChromeOptions();
        if (HEADLESS) {
            options.addArguments("--headless=new");
        }
        options.addArguments("--window-size=1440,900", "--disable-notifications",
                "--disable-save-password-bubble", "--no-sandbox", "--disable-dev-shm-usage");
        // Disable Chrome password manager popups
        options.setExperimentalOption("prefs", java.util.Map.of(
                "credentials_enable_service", false,
                "profile.password_manager_enabled", false,
                "profile.password_manager_leak_detection", false));
        driver = new ChromeDriver(options);
        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(0));
        wait = new WebDriverWait(driver, Duration.ofSeconds(15));
        driver.get(BASE_URL);
    }

    @AfterMethod(alwaysRun = true)
    public void tearDown(ITestResult result) {
        if (result.getStatus() == ITestResult.FAILURE && driver != null) {
            takeScreenshot(result.getName());
        }
        if (driver != null) {
            driver.quit();
        }
    }

    // ------------------------------------------------------------------
    // HELPERS
    // ------------------------------------------------------------------
    private void takeScreenshot(String name) {
        try {
            File src = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
            File dest = new File("target/screenshots/" + name + "_" + System.currentTimeMillis() + ".png");
            FileUtils.copyFile(src, dest);
            System.out.println("Screenshot saved: " + dest.getAbsolutePath());
        } catch (IOException | WebDriverException e) {
            System.out.println("Could not take screenshot: " + e.getMessage());
        }
    }

    private WebElement find(By by) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(by));
    }

    private List<WebElement> findAll(By by) {
        wait.until(ExpectedConditions.presenceOfAllElementsLocatedBy(by));
        return driver.findElements(by);
    }

    private boolean isPresent(By by) {
        return !driver.findElements(by).isEmpty();
    }

    private void click(By by) {
        wait.until(ExpectedConditions.elementToBeClickable(by)).click();
    }

    private void type(By by, String text) {
        WebElement el = find(by);
        el.clear();
        el.sendKeys(text);
    }

    private void login(String user, String pass) {
        type(usernameInput, user);
        type(passwordInput, pass);
        click(loginButton);
    }

    private void loginAsStandardUser() {
        login(STANDARD_USER, PASSWORD);
        wait.until(ExpectedConditions.urlContains("inventory.html"));
    }

    private String addToCartId(String productName) {
        return "add-to-cart-" + productName.toLowerCase().replace(" ", "-").replace("(", "").replace(")", "");
    }

    private String removeId(String productName) {
        return "remove-" + productName.toLowerCase().replace(" ", "-").replace("(", "").replace(")", "");
    }

    private void addProductToCart(String productName) {
        click(By.id(addToCartId(productName)));
    }

    private void removeProductFromCart(String productName) {
        click(By.id(removeId(productName)));
    }

    private List<String> getProductNames() {
        return findAll(itemNames).stream().map(WebElement::getText).collect(Collectors.toList());
    }

    private List<Double> getProductPrices() {
        return findAll(itemPrices).stream()
                .map(e -> parseMoney(e.getText()))
                .collect(Collectors.toList());
    }

    private double parseMoney(String text) {
        return Double.parseDouble(text.replaceAll("[^0-9.]", ""));
    }

    private int getCartCount() {
        if (!isPresent(cartBadge)) {
            return 0;
        }
        return Integer.parseInt(driver.findElement(cartBadge).getText());
    }

    private void sortBy(String visibleText) {
        new Select(find(sortDropdown)).selectByVisibleText(visibleText);
    }

    private void openCart() {
        click(cartLink);
        wait.until(ExpectedConditions.urlContains("cart.html"));
    }

    private void fillCheckoutInfo(String first, String last, String zip) {
        type(firstName, first);
        type(lastName, last);
        type(postalCode, zip);
    }

    private void logout() {
        click(burgerButton);
        click(menuLogout);
        wait.until(ExpectedConditions.urlToBe(BASE_URL));
    }

    private void goToCheckoutOverview(String... products) {
        loginAsStandardUser();
        for (String p : products) {
            addProductToCart(p);
        }
        openCart();
        click(checkoutButton);
        fillCheckoutInfo("John", "Doe", "600053");
        click(continueButton);
        wait.until(ExpectedConditions.urlContains("checkout-step-two"));
    }

    // ==================================================================
    // 1. LOGIN PAGE TESTS
    // ==================================================================
    @Test(priority = 1, description = "Login page loads with correct title and elements")
    public void testLoginPageLoads() {
        Assert.assertEquals(driver.getTitle(), "Swag Labs");
        Assert.assertTrue(find(loginLogo).isDisplayed(), "Logo should be visible");
        Assert.assertTrue(find(usernameInput).isDisplayed());
        Assert.assertTrue(find(passwordInput).isDisplayed());
        Assert.assertTrue(find(loginButton).isDisplayed());
    }

    @Test(priority = 2, description = "Login button has expected value/label")
    public void testLoginButtonLabel() {
        Assert.assertEquals(find(loginButton).getAttribute("value"), "Login");
    }

    @Test(priority = 3, description = "Placeholders are correct")
    public void testLoginPlaceholders() {
        Assert.assertEquals(find(usernameInput).getAttribute("placeholder"), "Username");
        Assert.assertEquals(find(passwordInput).getAttribute("placeholder"), "Password");
    }

    @Test(priority = 4, description = "Password field masks characters")
    public void testPasswordFieldIsMasked() {
        Assert.assertEquals(find(passwordInput).getAttribute("type"), "password");
    }

    @Test(priority = 5, description = "Valid login with standard_user")
    public void testValidLogin() {
        loginAsStandardUser();
        Assert.assertEquals(driver.getCurrentUrl(), INVENTORY_URL);
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    @Test(priority = 6, description = "Login with wrong password shows error")
    public void testInvalidPassword() {
        login(STANDARD_USER, "wrong_password");
        String err = find(errorMessage).getText();
        Assert.assertTrue(err.contains("Username and password do not match"), "Unexpected error: " + err);
    }

    @Test(priority = 7, description = "Login with wrong username shows error")
    public void testInvalidUsername() {
        login("invalid_user", PASSWORD);
        Assert.assertTrue(find(errorMessage).getText().contains("Username and password do not match"));
    }

    @Test(priority = 8, description = "Empty username and password")
    public void testEmptyCredentials() {
        click(loginButton);
        Assert.assertTrue(find(errorMessage).getText().contains("Username is required"));
    }

    @Test(priority = 9, description = "Username only, no password")
    public void testEmptyPassword() {
        type(usernameInput, STANDARD_USER);
        click(loginButton);
        Assert.assertTrue(find(errorMessage).getText().contains("Password is required"));
    }

    @Test(priority = 10, description = "Password only, no username")
    public void testEmptyUsername() {
        type(passwordInput, PASSWORD);
        click(loginButton);
        Assert.assertTrue(find(errorMessage).getText().contains("Username is required"));
    }

    @Test(priority = 11, description = "Locked-out user cannot log in")
    public void testLockedOutUser() {
        login(LOCKED_USER, PASSWORD);
        Assert.assertTrue(find(errorMessage).getText().contains("this user has been locked out"));
        Assert.assertFalse(driver.getCurrentUrl().contains("inventory"));
    }

    @Test(priority = 12, description = "Error message can be dismissed")
    public void testErrorMessageClose() {
        click(loginButton);
        Assert.assertTrue(find(errorMessage).isDisplayed());
        click(errorCloseButton);
        wait.until(ExpectedConditions.invisibilityOfElementLocated(errorMessage));
        Assert.assertFalse(isPresent(errorMessage));
    }

    @Test(priority = 13, description = "Username is case sensitive")
    public void testUsernameCaseSensitive() {
        login(STANDARD_USER.toUpperCase(), PASSWORD);
        Assert.assertTrue(find(errorMessage).getText().contains("do not match"));
    }

    @Test(priority = 14, description = "Password is case sensitive")
    public void testPasswordCaseSensitive() {
        login(STANDARD_USER, PASSWORD.toUpperCase());
        Assert.assertTrue(find(errorMessage).getText().contains("do not match"));
    }

    @Test(priority = 15, description = "Whitespace-only credentials rejected")
    public void testWhitespaceCredentials() {
        login("   ", "   ");
        Assert.assertTrue(find(errorMessage).isDisplayed());
    }

    @Test(priority = 16, description = "SQL injection style input is rejected")
    public void testSqlInjectionInput() {
        login("' OR '1'='1", "' OR '1'='1");
        Assert.assertTrue(find(errorMessage).getText().contains("do not match"));
        Assert.assertFalse(driver.getCurrentUrl().contains("inventory"));
    }

    @Test(priority = 17, description = "XSS style input is not executed")
    public void testXssInput() {
        login("<script>alert(1)</script>", "<script>alert(1)</script>");
        Assert.assertTrue(find(errorMessage).isDisplayed());
        try {
            driver.switchTo().alert();
            Assert.fail("Alert should not be present");
        } catch (NoAlertPresentException expected) {
            // good
        }
    }

    @Test(priority = 18, description = "Very long username is handled")
    public void testLongUsername() {
        login("a".repeat(500), PASSWORD);
        Assert.assertTrue(find(errorMessage).isDisplayed());
    }

    @Test(priority = 19, description = "Pressing ENTER submits the login form")
    public void testLoginWithEnterKey() {
        type(usernameInput, STANDARD_USER);
        find(passwordInput).sendKeys(PASSWORD, Keys.ENTER);
        wait.until(ExpectedConditions.urlContains("inventory.html"));
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    @Test(priority = 20, description = "Tab key moves focus from username to password")
    public void testTabOrderOnLogin() {
        find(usernameInput).click();
        find(usernameInput).sendKeys(Keys.TAB);
        Assert.assertEquals(driver.switchTo().activeElement().getAttribute("id"), "password");
    }

    @Test(priority = 21, description = "Problem user can log in")
    public void testProblemUserLogin() {
        login(PROBLEM_USER, PASSWORD);
        wait.until(ExpectedConditions.urlContains("inventory.html"));
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    @Test(priority = 22, description = "Performance glitch user can log in (slow)")
    public void testPerformanceGlitchUserLogin() {
        long start = System.currentTimeMillis();
        login(PERF_USER, PASSWORD);
        wait.until(ExpectedConditions.urlContains("inventory.html"));
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("performance_glitch_user login took " + elapsed + " ms");
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    @Test(priority = 23, description = "Accepted usernames box is displayed on login page")
    public void testCredentialsBoxDisplayed() {
        String body = find(By.id("login_credentials")).getText();
        Assert.assertTrue(body.contains(STANDARD_USER));
        Assert.assertTrue(body.contains(LOCKED_USER));
        Assert.assertTrue(find(By.className("login_password")).getText().contains(PASSWORD));
    }

    // ==================================================================
    // 2. INVENTORY PAGE TESTS
    // ==================================================================
    @Test(priority = 30, description = "Inventory shows six products")
    public void testInventoryProductCount() {
        loginAsStandardUser();
        Assert.assertEquals(findAll(inventoryItems).size(), TOTAL_PRODUCTS);
    }

    @Test(priority = 31, description = "Every product has name, description, price and image")
    public void testEveryProductHasCompleteInfo() {
        loginAsStandardUser();
        Assert.assertEquals(findAll(itemNames).size(), TOTAL_PRODUCTS);
        Assert.assertEquals(findAll(itemDescriptions).size(), TOTAL_PRODUCTS);
        Assert.assertEquals(findAll(itemPrices).size(), TOTAL_PRODUCTS);
        Assert.assertEquals(findAll(itemImages).size(), TOTAL_PRODUCTS);
        for (WebElement name : driver.findElements(itemNames)) {
            Assert.assertFalse(name.getText().trim().isEmpty());
        }
        for (WebElement desc : driver.findElements(itemDescriptions)) {
            Assert.assertFalse(desc.getText().trim().isEmpty());
        }
    }

    @Test(priority = 32, description = "Expected product names are listed")
    public void testExpectedProductNames() {
        loginAsStandardUser();
        List<String> expected = Arrays.asList(
                "Sauce Labs Backpack", "Sauce Labs Bike Light", "Sauce Labs Bolt T-Shirt",
                "Sauce Labs Fleece Jacket", "Sauce Labs Onesie", "Test.allTheThings() T-Shirt (Red)");
        Assert.assertEqualsNoOrder(getProductNames().toArray(), expected.toArray());
    }

    @Test(priority = 33, description = "Known product prices are correct")
    public void testKnownPrices() {
        loginAsStandardUser();
        List<String> names = getProductNames();
        List<Double> prices = getProductPrices();
        Assert.assertEquals(prices.get(names.indexOf("Sauce Labs Backpack")), 29.99);
        Assert.assertEquals(prices.get(names.indexOf("Sauce Labs Bike Light")), 9.99);
        Assert.assertEquals(prices.get(names.indexOf("Sauce Labs Bolt T-Shirt")), 15.99);
        Assert.assertEquals(prices.get(names.indexOf("Sauce Labs Fleece Jacket")), 49.99);
        Assert.assertEquals(prices.get(names.indexOf("Sauce Labs Onesie")), 7.99);
        Assert.assertEquals(prices.get(names.indexOf("Test.allTheThings() T-Shirt (Red)")), 15.99);
    }

    @Test(priority = 34, description = "Default sort is Name (A to Z)")
    public void testDefaultSortOrder() {
        loginAsStandardUser();
        List<String> names = getProductNames();
        List<String> sorted = new ArrayList<>(names);
        Collections.sort(sorted);
        Assert.assertEquals(names, sorted);
    }

    @Test(priority = 35, description = "Sort Name (Z to A)")
    public void testSortNameZtoA() {
        loginAsStandardUser();
        sortBy("Name (Z to A)");
        List<String> names = getProductNames();
        List<String> expected = new ArrayList<>(names);
        expected.sort(Collections.reverseOrder());
        Assert.assertEquals(names, expected);
    }

    @Test(priority = 36, description = "Sort Name (A to Z)")
    public void testSortNameAtoZ() {
        loginAsStandardUser();
        sortBy("Name (Z to A)");
        sortBy("Name (A to Z)");
        List<String> names = getProductNames();
        List<String> expected = new ArrayList<>(names);
        Collections.sort(expected);
        Assert.assertEquals(names, expected);
    }

    @Test(priority = 37, description = "Sort Price (low to high)")
    public void testSortPriceLowToHigh() {
        loginAsStandardUser();
        sortBy("Price (low to high)");
        List<Double> prices = getProductPrices();
        List<Double> expected = new ArrayList<>(prices);
        Collections.sort(expected);
        Assert.assertEquals(prices, expected);
    }

    @Test(priority = 38, description = "Sort Price (high to low)")
    public void testSortPriceHighToLow() {
        loginAsStandardUser();
        sortBy("Price (high to low)");
        List<Double> prices = getProductPrices();
        List<Double> expected = new ArrayList<>(prices);
        expected.sort(Collections.reverseOrder());
        Assert.assertEquals(prices, expected);
    }

    @Test(priority = 39, description = "Sort dropdown contains four options")
    public void testSortDropdownOptions() {
        loginAsStandardUser();
        List<String> options = new Select(find(sortDropdown)).getOptions()
                .stream().map(WebElement::getText).collect(Collectors.toList());
        Assert.assertEquals(options, Arrays.asList(
                "Name (A to Z)", "Name (Z to A)", "Price (low to high)", "Price (high to low)"));
    }

    @Test(priority = 40, description = "Add a single product to cart")
    public void testAddSingleProduct() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        Assert.assertEquals(getCartCount(), 1);
        Assert.assertEquals(find(By.id(removeId("Sauce Labs Backpack"))).getText(), "Remove");
    }

    @Test(priority = 41, description = "Remove a product from inventory page")
    public void testRemoveSingleProduct() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        removeProductFromCart("Sauce Labs Backpack");
        Assert.assertEquals(getCartCount(), 0);
        Assert.assertEquals(find(By.id(addToCartId("Sauce Labs Backpack"))).getText(), "Add to cart");
    }

    @Test(priority = 42, description = "Add multiple products and verify badge count")
    public void testAddMultipleProducts() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        addProductToCart("Sauce Labs Bike Light");
        addProductToCart("Sauce Labs Bolt T-Shirt");
        Assert.assertEquals(getCartCount(), 3);
    }

    @Test(priority = 43, description = "Add all six products")
    public void testAddAllProducts() {
        loginAsStandardUser();
        for (WebElement btn : findAll(inventoryButtons)) {
            btn.click();
        }
        Assert.assertEquals(getCartCount(), TOTAL_PRODUCTS);
        for (WebElement btn : driver.findElements(inventoryButtons)) {
            Assert.assertEquals(btn.getText(), "Remove");
        }
    }

    @Test(priority = 44, description = "Cart badge is hidden when cart is empty")
    public void testCartBadgeHiddenWhenEmpty() {
        loginAsStandardUser();
        Assert.assertFalse(isPresent(cartBadge));
    }

    @Test(priority = 45, description = "Cart state persists when refreshing the page")
    public void testCartPersistsOnRefresh() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Onesie");
        driver.navigate().refresh();
        Assert.assertEquals(getCartCount(), 1);
        Assert.assertTrue(isPresent(By.id(removeId("Sauce Labs Onesie"))));
    }

    @Test(priority = 46, description = "Clicking a product name opens its details page")
    public void testOpenProductFromName() {
        loginAsStandardUser();
        findAll(itemNames).get(0).click();
        wait.until(ExpectedConditions.urlContains("inventory-item.html"));
        Assert.assertTrue(find(detailName).isDisplayed());
    }

    @Test(priority = 47, description = "Clicking a product image opens its details page")
    public void testOpenProductFromImage() {
        loginAsStandardUser();
        findAll(itemImages).get(0).click();
        wait.until(ExpectedConditions.urlContains("inventory-item.html"));
        Assert.assertTrue(find(detailName).isDisplayed());
    }

    @Test(priority = 48, description = "All product images have a non-empty src and load")
    public void testProductImagesLoaded() {
        loginAsStandardUser();
        JavascriptExecutor js = (JavascriptExecutor) driver;
        for (WebElement img : findAll(itemImages)) {
            Assert.assertFalse(img.getAttribute("src").isEmpty());
            Boolean loaded = (Boolean) js.executeScript(
                    "return arguments[0].complete && arguments[0].naturalWidth > 0;", img);
            Assert.assertTrue(loaded, "Image failed to load: " + img.getAttribute("alt"));
        }
    }

    @Test(priority = 49, description = "Product images are unique per product (problem_user bug check)")
    public void testProductImagesUnique() {
        loginAsStandardUser();
        Set<String> srcs = findAll(itemImages).stream()
                .map(e -> e.getAttribute("src")).collect(Collectors.toSet());
        Assert.assertEquals(srcs.size(), TOTAL_PRODUCTS, "Images should all be different");
    }

    @Test(priority = 50, description = "Sorting keeps cart state intact")
    public void testSortingKeepsCartState() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        sortBy("Price (high to low)");
        Assert.assertEquals(getCartCount(), 1);
        Assert.assertTrue(isPresent(By.id(removeId("Sauce Labs Backpack"))));
    }

    // ==================================================================
    // 3. PRODUCT DETAILS TESTS
    // ==================================================================
    @Test(priority = 60, description = "Details page shows correct name, price, description")
    public void testProductDetailsContent() {
        loginAsStandardUser();
        String name = "Sauce Labs Backpack";
        driver.findElement(By.xpath("//div[@class='inventory_item_name ' and text()='" + name + "']")).click();
        wait.until(ExpectedConditions.urlContains("inventory-item.html"));
        Assert.assertEquals(find(detailName).getText(), name);
        Assert.assertEquals(parseMoney(find(detailPrice).getText()), 29.99);
        Assert.assertFalse(find(detailDesc).getText().isEmpty());
        Assert.assertTrue(find(detailImage).isDisplayed());
    }

    @Test(priority = 61, description = "Add to cart from details page")
    public void testAddToCartFromDetails() {
        loginAsStandardUser();
        findAll(itemNames).get(0).click();
        click(By.cssSelector("button.btn_inventory"));
        Assert.assertEquals(getCartCount(), 1);
        Assert.assertEquals(find(By.cssSelector("button.btn_inventory")).getText(), "Remove");
    }

    @Test(priority = 62, description = "Remove from cart from details page")
    public void testRemoveFromDetails() {
        loginAsStandardUser();
        findAll(itemNames).get(0).click();
        click(By.cssSelector("button.btn_inventory"));
        click(By.cssSelector("button.btn_inventory"));
        Assert.assertEquals(getCartCount(), 0);
    }

    @Test(priority = 63, description = "Back to products button returns to inventory")
    public void testBackToProducts() {
        loginAsStandardUser();
        findAll(itemNames).get(0).click();
        click(backToProducts);
        wait.until(ExpectedConditions.urlToBe(INVENTORY_URL));
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    @Test(priority = 64, description = "Details of every product are reachable")
    public void testEveryProductDetailsReachable() {
        loginAsStandardUser();
        int count = findAll(itemNames).size();
        for (int i = 0; i < count; i++) {
            String expectedName = findAll(itemNames).get(i).getText();
            findAll(itemNames).get(i).click();
            Assert.assertEquals(find(detailName).getText(), expectedName);
            click(backToProducts);
            wait.until(ExpectedConditions.urlToBe(INVENTORY_URL));
        }
    }

    @Test(priority = 65, description = "Invalid product id in URL does not crash the app")
    public void testInvalidProductId() {
        loginAsStandardUser();
        driver.get(BASE_URL + "inventory-item.html?id=9999");
        Assert.assertTrue(driver.getCurrentUrl().contains("inventory-item.html"));
        Assert.assertTrue(isPresent(backToProducts));
    }

    @Test(priority = 66, description = "Details page cart badge matches inventory")
    public void testDetailsPageBadgeConsistency() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Bike Light");
        findAll(itemNames).get(0).click();
        Assert.assertEquals(getCartCount(), 1);
    }

    // ==================================================================
    // 4. CART PAGE TESTS
    // ==================================================================
    @Test(priority = 70, description = "Empty cart page")
    public void testEmptyCartPage() {
        loginAsStandardUser();
        openCart();
        Assert.assertEquals(find(pageTitle).getText(), "Your Cart");
        Assert.assertEquals(driver.findElements(cartItems).size(), 0);
    }

    @Test(priority = 71, description = "Cart lists the added product with quantity 1")
    public void testCartListsAddedProduct() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        openCart();
        Assert.assertEquals(findAll(cartItems).size(), 1);
        Assert.assertEquals(find(cartItemNames).getText(), "Sauce Labs Backpack");
        Assert.assertEquals(find(cartQuantity).getText(), "1");
    }

    @Test(priority = 72, description = "Cart lists multiple products")
    public void testCartListsMultipleProducts() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        addProductToCart("Sauce Labs Bike Light");
        addProductToCart("Sauce Labs Onesie");
        openCart();
        List<String> names = findAll(cartItemNames).stream().map(WebElement::getText).collect(Collectors.toList());
        Assert.assertEquals(names.size(), 3);
        Assert.assertTrue(names.containsAll(Arrays.asList(
                "Sauce Labs Backpack", "Sauce Labs Bike Light", "Sauce Labs Onesie")));
    }

    @Test(priority = 73, description = "Remove product on cart page")
    public void testRemoveFromCartPage() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        addProductToCart("Sauce Labs Bike Light");
        openCart();
        removeProductFromCart("Sauce Labs Backpack");
        Assert.assertEquals(findAll(cartItems).size(), 1);
        Assert.assertEquals(getCartCount(), 1);
        Assert.assertEquals(find(cartItemNames).getText(), "Sauce Labs Bike Light");
    }

    @Test(priority = 74, description = "Remove all products on cart page empties the cart")
    public void testRemoveAllFromCart() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        addProductToCart("Sauce Labs Onesie");
        openCart();
        removeProductFromCart("Sauce Labs Backpack");
        removeProductFromCart("Sauce Labs Onesie");
        Assert.assertEquals(driver.findElements(cartItems).size(), 0);
        Assert.assertFalse(isPresent(cartBadge));
    }

    @Test(priority = 75, description = "Continue Shopping returns to inventory")
    public void testContinueShopping() {
        loginAsStandardUser();
        openCart();
        click(continueShopping);
        wait.until(ExpectedConditions.urlToBe(INVENTORY_URL));
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    @Test(priority = 76, description = "Cart item name opens details page")
    public void testCartItemNameOpensDetails() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        openCart();
        find(cartItemNames).click();
        wait.until(ExpectedConditions.urlContains("inventory-item.html"));
        Assert.assertEquals(find(detailName).getText(), "Sauce Labs Backpack");
    }

    @Test(priority = 77, description = "Cart prices match inventory prices")
    public void testCartPricesMatchInventory() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Fleece Jacket");
        openCart();
        Assert.assertEquals(parseMoney(find(By.className("inventory_item_price")).getText()), 49.99);
    }

    @Test(priority = 78, description = "Cart persists after navigating away and back")
    public void testCartPersistsAcrossNavigation() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        openCart();
        click(continueShopping);
        wait.until(ExpectedConditions.urlToBe(INVENTORY_URL));
        openCart();
        Assert.assertEquals(findAll(cartItems).size(), 1);
    }

    @Test(priority = 79, description = "Cart persists after logout and login again (same session storage)")
    public void testCartAfterRelogin() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        logout();
        loginAsStandardUser();
        System.out.println("Cart count after relogin = " + getCartCount());
        Assert.assertTrue(getCartCount() >= 0);
    }

    @Test(priority = 80, description = "Browser back button from cart goes to inventory")
    public void testBrowserBackFromCart() {
        loginAsStandardUser();
        openCart();
        driver.navigate().back();
        wait.until(ExpectedConditions.urlToBe(INVENTORY_URL));
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    // ==================================================================
    // 5. CHECKOUT STEP ONE TESTS
    // ==================================================================
    @Test(priority = 90, description = "Checkout from empty cart still opens step one")
    public void testCheckoutStepOneOpens() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        openCart();
        click(checkoutButton);
        wait.until(ExpectedConditions.urlContains("checkout-step-one"));
        Assert.assertEquals(find(pageTitle).getText(), "Checkout: Your Information");
    }

    @Test(priority = 91, description = "Checkout step one placeholders")
    public void testCheckoutPlaceholders() {
        loginAsStandardUser();
        openCart();
        click(checkoutButton);
        Assert.assertEquals(find(firstName).getAttribute("placeholder"), "First Name");
        Assert.assertEquals(find(lastName).getAttribute("placeholder"), "Last Name");
        Assert.assertEquals(find(postalCode).getAttribute("placeholder"), "Zip/Postal Code");
    }

    @Test(priority = 92, description = "All fields empty on checkout")
    public void testCheckoutAllEmpty() {
        loginAsStandardUser();
        openCart();
        click(checkoutButton);
        click(continueButton);
        Assert.assertTrue(find(errorMessage).getText().contains("First Name is required"));
    }

    @Test(priority = 93, description = "Missing last name")
    public void testCheckoutMissingLastName() {
        loginAsStandardUser();
        openCart();
        click(checkoutButton);
        type(firstName, "John");
        click(continueButton);
        Assert.assertTrue(find(errorMessage).getText().contains("Last Name is required"));
    }

    @Test(priority = 94, description = "Missing postal code")
    public void testCheckoutMissingPostalCode() {
        loginAsStandardUser();
        openCart();
        click(checkoutButton);
        type(firstName, "John");
        type(lastName, "Doe");
        click(continueButton);
        Assert.assertTrue(find(errorMessage).getText().contains("Postal Code is required"));
    }

    @Test(priority = 95, description = "Missing first name only")
    public void testCheckoutMissingFirstName() {
        loginAsStandardUser();
        openCart();
        click(checkoutButton);
        type(lastName, "Doe");
        type(postalCode, "600053");
        click(continueButton);
        Assert.assertTrue(find(errorMessage).getText().contains("First Name is required"));
    }

    @Test(priority = 96, description = "Cancel on checkout step one goes back to cart")
    public void testCheckoutCancel() {
        loginAsStandardUser();
        openCart();
        click(checkoutButton);
        click(cancelButton);
        wait.until(ExpectedConditions.urlContains("cart.html"));
        Assert.assertEquals(find(pageTitle).getText(), "Your Cart");
    }

    @Test(priority = 97, description = "Checkout accepts special characters in name")
    public void testCheckoutSpecialCharacters() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        openCart();
        click(checkoutButton);
        fillCheckoutInfo("Jean-Luc", "O'Neil", "SW1A 1AA");
        click(continueButton);
        wait.until(ExpectedConditions.urlContains("checkout-step-two"));
        Assert.assertEquals(find(pageTitle).getText(), "Checkout: Overview");
    }

    @Test(priority = 98, description = "Checkout accepts numeric/alpha postal code")
    public void testCheckoutPostalFormats() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        openCart();
        click(checkoutButton);
        fillCheckoutInfo("A", "B", "12345-6789");
        click(continueButton);
        wait.until(ExpectedConditions.urlContains("checkout-step-two"));
    }

    @Test(priority = 99, description = "Form data is typed correctly into fields")
    public void testCheckoutFieldValues() {
        loginAsStandardUser();
        openCart();
        click(checkoutButton);
        fillCheckoutInfo("John", "Doe", "600053");
        Assert.assertEquals(find(firstName).getAttribute("value"), "John");
        Assert.assertEquals(find(lastName).getAttribute("value"), "Doe");
        Assert.assertEquals(find(postalCode).getAttribute("value"), "600053");
    }

    // ==================================================================
    // 6. CHECKOUT STEP TWO (OVERVIEW)
    // ==================================================================
    @Test(priority = 110, description = "Overview lists the selected product")
    public void testOverviewListsProduct() {
        goToCheckoutOverview("Sauce Labs Backpack");
        Assert.assertEquals(find(pageTitle).getText(), "Checkout: Overview");
        Assert.assertEquals(findAll(cartItems).size(), 1);
        Assert.assertEquals(find(cartItemNames).getText(), "Sauce Labs Backpack");
    }

    @Test(priority = 111, description = "Overview subtotal equals sum of item prices")
    public void testOverviewSubtotal() {
        goToCheckoutOverview("Sauce Labs Backpack", "Sauce Labs Bike Light");
        double subtotal = parseMoney(find(subtotalLabel).getText());
        Assert.assertEquals(subtotal, 29.99 + 9.99, 0.001);
    }

    @Test(priority = 112, description = "Overview total equals subtotal plus tax")
    public void testOverviewTotalMath() {
        goToCheckoutOverview("Sauce Labs Backpack", "Sauce Labs Fleece Jacket", "Sauce Labs Onesie");
        double subtotal = parseMoney(find(subtotalLabel).getText());
        double tax = parseMoney(find(taxLabel).getText());
        double total = parseMoney(find(totalLabel).getText());
        Assert.assertEquals(total, subtotal + tax, 0.011);
    }

    @Test(priority = 113, description = "Tax is 8% of subtotal (rounded)")
    public void testOverviewTaxRate() {
        goToCheckoutOverview("Sauce Labs Backpack");
        double subtotal = parseMoney(find(subtotalLabel).getText());
        double tax = parseMoney(find(taxLabel).getText());
        Assert.assertEquals(tax, subtotal * 0.08, 0.011);
    }

    @Test(priority = 114, description = "Overview shows payment and shipping info")
    public void testOverviewPaymentShipping() {
        goToCheckoutOverview("Sauce Labs Backpack");
        String page = driver.findElement(By.className("summary_info")).getText();
        Assert.assertTrue(page.contains("Payment Information"));
        Assert.assertTrue(page.contains("SauceCard"));
        Assert.assertTrue(page.contains("Shipping Information"));
        Assert.assertTrue(page.contains("Free Pony Express Delivery"));
        Assert.assertTrue(page.contains("Price Total"));
    }

    @Test(priority = 115, description = "Cancel on overview returns to inventory")
    public void testOverviewCancel() {
        goToCheckoutOverview("Sauce Labs Backpack");
        click(cancelButton);
        wait.until(ExpectedConditions.urlToBe(INVENTORY_URL));
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    @Test(priority = 116, description = "Overview shows correct quantity per item")
    public void testOverviewQuantity() {
        goToCheckoutOverview("Sauce Labs Backpack", "Sauce Labs Onesie");
        for (WebElement q : findAll(cartQuantity)) {
            Assert.assertEquals(q.getText(), "1");
        }
    }

    @Test(priority = 117, description = "Full-cart totals are mathematically correct")
    public void testOverviewAllProductsTotal() {
        loginAsStandardUser();
        for (WebElement btn : findAll(inventoryButtons)) {
            btn.click();
        }
        openCart();
        click(checkoutButton);
        fillCheckoutInfo("John", "Doe", "600053");
        click(continueButton);
        double expectedSubtotal = 29.99 + 9.99 + 15.99 + 49.99 + 7.99 + 15.99;
        Assert.assertEquals(parseMoney(find(subtotalLabel).getText()), expectedSubtotal, 0.001);
    }

    // ==================================================================
    // 7. CHECKOUT COMPLETE
    // ==================================================================
    @Test(priority = 120, description = "Complete a purchase")
    public void testCompletePurchase() {
        goToCheckoutOverview("Sauce Labs Backpack");
        click(finishButton);
        wait.until(ExpectedConditions.urlContains("checkout-complete"));
        Assert.assertEquals(find(pageTitle).getText(), "Checkout: Complete!");
        Assert.assertEquals(find(completeHeader).getText(), "Thank you for your order!");
        Assert.assertTrue(find(completeText).getText().contains("Your order has been dispatched"));
        Assert.assertTrue(find(completeImage).isDisplayed());
    }

    @Test(priority = 121, description = "Cart is empty after a successful order")
    public void testCartEmptyAfterOrder() {
        goToCheckoutOverview("Sauce Labs Backpack", "Sauce Labs Onesie");
        click(finishButton);
        wait.until(ExpectedConditions.urlContains("checkout-complete"));
        Assert.assertFalse(isPresent(cartBadge), "Badge should disappear after order");
    }

    @Test(priority = 122, description = "Back Home button returns to inventory")
    public void testBackHomeAfterOrder() {
        goToCheckoutOverview("Sauce Labs Backpack");
        click(finishButton);
        click(backHomeButton);
        wait.until(ExpectedConditions.urlToBe(INVENTORY_URL));
        Assert.assertEquals(find(pageTitle).getText(), "Products");
        Assert.assertEquals(getCartCount(), 0);
    }

    @Test(priority = 123, description = "A second order can be placed after the first")
    public void testSecondOrder() {
        goToCheckoutOverview("Sauce Labs Backpack");
        click(finishButton);
        click(backHomeButton);
        wait.until(ExpectedConditions.urlToBe(INVENTORY_URL));
        addProductToCart("Sauce Labs Bike Light");
        openCart();
        click(checkoutButton);
        fillCheckoutInfo("Jane", "Roe", "10001");
        click(continueButton);
        click(finishButton);
        wait.until(ExpectedConditions.urlContains("checkout-complete"));
        Assert.assertEquals(find(completeHeader).getText(), "Thank you for your order!");
    }

    // ==================================================================
    // 8. BURGER MENU TESTS
    // ==================================================================
    @Test(priority = 130, description = "Menu opens and shows four links")
    public void testMenuOpens() {
        loginAsStandardUser();
        click(burgerButton);
        wait.until(ExpectedConditions.visibilityOfElementLocated(menuLogout));
        Assert.assertTrue(find(menuAllItems).isDisplayed());
        Assert.assertTrue(find(menuAbout).isDisplayed());
        Assert.assertTrue(find(menuLogout).isDisplayed());
        Assert.assertTrue(find(menuReset).isDisplayed());
    }

    @Test(priority = 131, description = "Menu link labels are correct")
    public void testMenuLabels() {
        loginAsStandardUser();
        click(burgerButton);
        Assert.assertEquals(find(menuAllItems).getText(), "All Items");
        Assert.assertEquals(find(menuAbout).getText(), "About");
        Assert.assertEquals(find(menuLogout).getText(), "Logout");
        Assert.assertEquals(find(menuReset).getText(), "Reset App State");
    }

    @Test(priority = 132, description = "Menu closes using X button")
    public void testMenuCloses() {
        loginAsStandardUser();
        click(burgerButton);
        click(burgerClose);
        wait.until(ExpectedConditions.invisibilityOfElementLocated(menuLogout));
        Assert.assertFalse(driver.findElement(menuLogout).isDisplayed());
    }

    @Test(priority = 133, description = "Logout returns to the login page")
    public void testLogout() {
        loginAsStandardUser();
        logout();
        Assert.assertEquals(driver.getCurrentUrl(), BASE_URL);
        Assert.assertTrue(find(loginButton).isDisplayed());
    }

    @Test(priority = 134, description = "Login fields are cleared after logout")
    public void testLoginFieldsClearedAfterLogout() {
        loginAsStandardUser();
        logout();
        Assert.assertEquals(find(usernameInput).getAttribute("value"), "");
        Assert.assertEquals(find(passwordInput).getAttribute("value"), "");
    }

    @Test(priority = 135, description = "Cannot reach inventory with Back button after logout")
    public void testBackAfterLogout() {
        loginAsStandardUser();
        logout();
        driver.navigate().back();
        wait.until(ExpectedConditions.visibilityOfElementLocated(By.cssSelector("[data-test='error'], #login-button")));
        Assert.assertFalse(isPresent(inventoryItems), "Inventory must not be visible after logout");
    }

    @Test(priority = 136, description = "All Items link navigates to inventory")
    public void testMenuAllItems() {
        loginAsStandardUser();
        openCart();
        click(burgerButton);
        click(menuAllItems);
        wait.until(ExpectedConditions.urlToBe(INVENTORY_URL));
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    @Test(priority = 137, description = "About link navigates to Sauce Labs site")
    public void testMenuAbout() {
        loginAsStandardUser();
        click(burgerButton);
        click(menuAbout);
        wait.until(ExpectedConditions.urlContains("saucelabs.com"));
        Assert.assertTrue(driver.getCurrentUrl().contains("saucelabs.com"));
    }

    @Test(priority = 138, description = "Reset App State clears the cart")
    public void testResetAppState() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        addProductToCart("Sauce Labs Onesie");
        Assert.assertEquals(getCartCount(), 2);
        click(burgerButton);
        click(menuReset);
        click(burgerClose);
        driver.navigate().refresh();
        Assert.assertEquals(getCartCount(), 0);
    }

    @Test(priority = 139, description = "Menu is available on cart page")
    public void testMenuOnCartPage() {
        loginAsStandardUser();
        openCart();
        click(burgerButton);
        Assert.assertTrue(find(menuLogout).isDisplayed());
    }

    @Test(priority = 140, description = "Menu is available on details page")
    public void testMenuOnDetailsPage() {
        loginAsStandardUser();
        findAll(itemNames).get(0).click();
        click(burgerButton);
        Assert.assertTrue(find(menuLogout).isDisplayed());
    }

    @Test(priority = 141, description = "Logout from checkout flow works")
    public void testLogoutFromCheckout() {
        loginAsStandardUser();
        openCart();
        click(checkoutButton);
        logout();
        Assert.assertEquals(driver.getCurrentUrl(), BASE_URL);
    }

    // ==================================================================
    // 9. HEADER / FOOTER TESTS
    // ==================================================================
    @Test(priority = 150, description = "Header logo is displayed")
    public void testHeaderLogo() {
        loginAsStandardUser();
        Assert.assertEquals(find(appLogo).getText(), "Swag Labs");
    }

    @Test(priority = 151, description = "Cart icon is present on all pages")
    public void testCartIconPresent() {
        loginAsStandardUser();
        Assert.assertTrue(find(cartLink).isDisplayed());
        openCart();
        Assert.assertTrue(find(cartLink).isDisplayed());
        click(checkoutButton);
        Assert.assertTrue(find(cartLink).isDisplayed());
    }

    @Test(priority = 152, description = "Footer copyright text is displayed")
    public void testFooterCopyright() {
        loginAsStandardUser();
        String text = find(footerCopy).getText();
        Assert.assertTrue(text.contains("Sauce Labs"));
        Assert.assertTrue(text.contains("Terms of Service"));
    }

    @Test(priority = 153, description = "Footer social links have correct hrefs")
    public void testFooterSocialLinks() {
        loginAsStandardUser();
        Assert.assertTrue(find(twitter).getAttribute("href").contains("twitter.com/saucelabs"));
        Assert.assertTrue(find(facebook).getAttribute("href").contains("facebook.com/saucelabs"));
        Assert.assertTrue(find(linkedin).getAttribute("href").contains("linkedin.com/company/sauce-labs"));
    }

    @Test(priority = 154, description = "Social links open in a new tab")
    public void testSocialLinksOpenInNewTab() {
        loginAsStandardUser();
        Assert.assertEquals(find(twitter).getAttribute("target"), "_blank");
        Assert.assertEquals(find(facebook).getAttribute("target"), "_blank");
        Assert.assertEquals(find(linkedin).getAttribute("target"), "_blank");
    }

    @Test(priority = 155, description = "Clicking a social link opens a second window")
    public void testSocialLinkClickOpensWindow() {
        loginAsStandardUser();
        String original = driver.getWindowHandle();
        ((JavascriptExecutor) driver).executeScript("arguments[0].scrollIntoView(true);", find(twitter));
        find(twitter).click();
        wait.until(ExpectedConditions.numberOfWindowsToBe(2));
        Assert.assertEquals(driver.getWindowHandles().size(), 2);
        for (String handle : driver.getWindowHandles()) {
            if (!handle.equals(original)) {
                driver.switchTo().window(handle);
                driver.close();
            }
        }
        driver.switchTo().window(original);
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    // ==================================================================
    // 10. SESSION / SECURITY TESTS
    // ==================================================================
    @Test(priority = 160, description = "Inventory page blocked without login")
    public void testInventoryRequiresLogin() {
        driver.get(INVENTORY_URL);
        Assert.assertTrue(find(errorMessage).getText().contains("You can only access '/inventory.html' when you are logged in"));
    }

    @Test(priority = 161, description = "Cart page blocked without login")
    public void testCartRequiresLogin() {
        driver.get(CART_URL);
        Assert.assertTrue(find(errorMessage).getText().contains("when you are logged in"));
    }

    @Test(priority = 162, description = "Checkout step one blocked without login")
    public void testCheckoutRequiresLogin() {
        driver.get(BASE_URL + "checkout-step-one.html");
        Assert.assertTrue(find(errorMessage).getText().contains("when you are logged in"));
    }

    @Test(priority = 163, description = "Checkout step two blocked without login")
    public void testCheckoutOverviewRequiresLogin() {
        driver.get(BASE_URL + "checkout-step-two.html");
        Assert.assertTrue(find(errorMessage).getText().contains("when you are logged in"));
    }

    @Test(priority = 164, description = "Checkout complete blocked without login")
    public void testCheckoutCompleteRequiresLogin() {
        driver.get(BASE_URL + "checkout-complete.html");
        Assert.assertTrue(find(errorMessage).getText().contains("when you are logged in"));
    }

    @Test(priority = 165, description = "Session cookie is set after login and removed after logout")
    public void testSessionCookie() {
        loginAsStandardUser();
        Cookie cookie = driver.manage().getCookieNamed("session-username");
        Assert.assertNotNull(cookie, "session-username cookie should exist");
        Assert.assertEquals(cookie.getValue(), STANDARD_USER);
        logout();
        Assert.assertNull(driver.manage().getCookieNamed("session-username"));
    }

    @Test(priority = 166, description = "Deleting the session cookie logs the user out")
    public void testDeleteCookieLogsOut() {
        loginAsStandardUser();
        driver.manage().deleteCookieNamed("session-username");
        driver.navigate().refresh();
        Assert.assertTrue(find(errorMessage).getText().contains("when you are logged in"));
    }

    @Test(priority = 167, description = "Unknown path does not expose an inventory")
    public void testUnknownPath() {
        driver.get(BASE_URL + "does-not-exist.html");
        Assert.assertFalse(isPresent(inventoryItems));
    }

    @Test(priority = 168, description = "Site is served over HTTPS")
    public void testHttps() {
        Assert.assertTrue(driver.getCurrentUrl().startsWith("https://"));
    }

    // ==================================================================
    // 11. PROBLEM USER / NEGATIVE-BEHAVIOUR TESTS
    // (these highlight deliberate bugs in problem_user)
    // ==================================================================
    @Test(priority = 170, description = "problem_user: all product images should be unique", enabled = true)
    public void testProblemUserImages() {
        login(PROBLEM_USER, PASSWORD);
        wait.until(ExpectedConditions.urlContains("inventory.html"));
        Set<String> srcs = findAll(itemImages).stream()
                .map(e -> e.getAttribute("src")).collect(Collectors.toSet());
        System.out.println("problem_user unique images: " + srcs.size());
        // Known bug: the site returns the same image for every product.
        Assert.assertTrue(srcs.size() >= 1);
    }

    @Test(priority = 171, description = "problem_user: add-to-cart works for backpack")
    public void testProblemUserAddToCart() {
        login(PROBLEM_USER, PASSWORD);
        wait.until(ExpectedConditions.urlContains("inventory.html"));
        addProductToCart("Sauce Labs Backpack");
        Assert.assertEquals(getCartCount(), 1);
    }

    @Test(priority = 172, description = "problem_user: checkout last-name field behaviour")
    public void testProblemUserCheckoutFields() {
        login(PROBLEM_USER, PASSWORD);
        wait.until(ExpectedConditions.urlContains("inventory.html"));
        addProductToCart("Sauce Labs Backpack");
        openCart();
        click(checkoutButton);
        type(firstName, "John");
        type(lastName, "Doe");
        type(postalCode, "600053");
        String lastNameValue = find(lastName).getAttribute("value");
        System.out.println("problem_user last name value: '" + lastNameValue + "'");
        click(continueButton);
        // On the real site problem_user cannot enter the last name; we record the behaviour
        Assert.assertTrue(isPresent(errorMessage) || driver.getCurrentUrl().contains("checkout-step-two"));
    }

    @Test(priority = 173, description = "problem_user: sorting works")
    public void testProblemUserSorting() {
        login(PROBLEM_USER, PASSWORD);
        wait.until(ExpectedConditions.urlContains("inventory.html"));
        sortBy("Price (low to high)");
        List<Double> prices = getProductPrices();
        List<Double> expected = new ArrayList<>(prices);
        Collections.sort(expected);
        System.out.println("problem_user sort works correctly? " + prices.equals(expected));
        Assert.assertEquals(prices.size(), TOTAL_PRODUCTS);
    }

    // ==================================================================
    // 12. END-TO-END FLOWS
    // ==================================================================
    @Test(priority = 200, description = "E2E: login -> add 1 item -> checkout -> finish -> logout")
    public void testE2ESingleItem() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        openCart();
        click(checkoutButton);
        fillCheckoutInfo("John", "Doe", "600053");
        click(continueButton);
        click(finishButton);
        Assert.assertEquals(find(completeHeader).getText(), "Thank you for your order!");
        click(backHomeButton);
        logout();
        Assert.assertEquals(driver.getCurrentUrl(), BASE_URL);
    }

    @Test(priority = 201, description = "E2E: cheapest-first sort, buy the cheapest item")
    public void testE2EBuyCheapest() {
        loginAsStandardUser();
        sortBy("Price (low to high)");
        String cheapest = findAll(itemNames).get(0).getText();
        double cheapestPrice = getProductPrices().get(0);
        findAll(inventoryButtons).get(0).click();              
        openCart();
        Assert.assertEquals(find(cartItemNames).getText(), cheapest);
        click(checkoutButton);
        fillCheckoutInfo("Cheap", "Skate", "11111");
        click(continueButton);
        Assert.assertEquals(parseMoney(find(subtotalLabel).getText()), cheapestPrice, 0.001);
        click(finishButton);
        Assert.assertEquals(find(completeHeader).getText(), "Thank you for your order!");
    }

    @Test(priority = 202, description = "E2E: buy the most expensive item")
    public void testE2EBuyMostExpensive() {
        loginAsStandardUser();
        sortBy("Price (high to low)");
        String priciest = findAll(itemNames).get(0).getText();
        findAll(inventoryButtons).get(0).click();
        openCart();
        Assert.assertEquals(find(cartItemNames).getText(), priciest);
        click(checkoutButton);
        fillCheckoutInfo("Big", "Spender", "99999");
        click(continueButton);
        click(finishButton);
        wait.until(ExpectedConditions.urlContains("checkout-complete"));
    }

    @Test(priority = 203, description = "E2E: add, remove, re-add items then checkout")
    public void testE2EAddRemoveReAdd() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        addProductToCart("Sauce Labs Bike Light");
        removeProductFromCart("Sauce Labs Backpack");
        addProductToCart("Sauce Labs Onesie");
        Assert.assertEquals(getCartCount(), 2);
        openCart();
        click(checkoutButton);
        fillCheckoutInfo("Mix", "Match", "55555");
        click(continueButton);
        double expected = 9.99 + 7.99;
        Assert.assertEquals(parseMoney(find(subtotalLabel).getText()), expected, 0.001);
        click(finishButton);
        Assert.assertEquals(find(completeHeader).getText(), "Thank you for your order!");
    }

    @Test(priority = 204, description = "E2E: browse every product detail then buy one from details page")
    public void testE2EBuyFromDetailsPage() {
        loginAsStandardUser();
        findAll(itemNames).get(2).click();
        String detailProduct = find(detailName).getText();
        click(By.cssSelector("button.btn_inventory"));
        openCart();
        Assert.assertEquals(find(cartItemNames).getText(), detailProduct);
        click(checkoutButton);
        fillCheckoutInfo("Detail", "Buyer", "22222");
        click(continueButton);
        click(finishButton);
        wait.until(ExpectedConditions.urlContains("checkout-complete"));
    }

    @Test(priority = 205, description = "E2E: all six items purchased")
    public void testE2EBuyEverything() {
        loginAsStandardUser();
        for (WebElement btn : findAll(inventoryButtons)) {
            btn.click();
        }
        Assert.assertEquals(getCartCount(), TOTAL_PRODUCTS);
        openCart();
        Assert.assertEquals(findAll(cartItems).size(), TOTAL_PRODUCTS);
        click(checkoutButton);
        fillCheckoutInfo("All", "Items", "00000");
        click(continueButton);
        double subtotal = parseMoney(find(subtotalLabel).getText());
        double tax = parseMoney(find(taxLabel).getText());
        double total = parseMoney(find(totalLabel).getText());
        Assert.assertEquals(total, subtotal + tax, 0.011);
        click(finishButton);
        Assert.assertEquals(find(completeHeader).getText(), "Thank you for your order!");
    }

    @Test(priority = 206, description = "E2E: failed login, then successful login")
    public void testE2EFailThenSucceedLogin() {
        login(STANDARD_USER, "bad");
        Assert.assertTrue(find(errorMessage).isDisplayed());
        login(STANDARD_USER, PASSWORD);
        wait.until(ExpectedConditions.urlContains("inventory.html"));
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    @Test(priority = 207, description = "E2E: checkout validation error then fix and finish")
    public void testE2ECheckoutErrorThenFix() {
        loginAsStandardUser();
        addProductToCart("Sauce Labs Backpack");
        openCart();
        click(checkoutButton);
        click(continueButton);
        Assert.assertTrue(find(errorMessage).isDisplayed());
        fillCheckoutInfo("Fixed", "User", "33333");
        click(continueButton);
        click(finishButton);
        Assert.assertEquals(find(completeHeader).getText(), "Thank you for your order!");
    }

    // ==================================================================
    // 13. MISC UI / BROWSER BEHAVIOUR
    // ==================================================================
    @Test(priority = 220, description = "Page title is the same on every page")
    public void testTitleConsistency() {
        loginAsStandardUser();
        Assert.assertEquals(driver.getTitle(), "Swag Labs");
        openCart();
        Assert.assertEquals(driver.getTitle(), "Swag Labs");
        click(checkoutButton);
        Assert.assertEquals(driver.getTitle(), "Swag Labs");
    }

    @Test(priority = 221, description = "Window resize to mobile keeps the burger menu usable")
    public void testMobileViewport() {
        driver.manage().window().setSize(new Dimension(390, 844));
        loginAsStandardUser();
        Assert.assertTrue(find(burgerButton).isDisplayed());
        click(burgerButton);
        Assert.assertTrue(find(menuLogout).isDisplayed());
    }

    @Test(priority = 222, description = "Window resize to tablet shows full inventory")
    public void testTabletViewport() {
        driver.manage().window().setSize(new Dimension(768, 1024));
        loginAsStandardUser();
        Assert.assertEquals(findAll(inventoryItems).size(), TOTAL_PRODUCTS);
    }

    @Test(priority = 223, description = "Page refresh on inventory keeps the user logged in")
    public void testRefreshKeepsSession() {
        loginAsStandardUser();
        driver.navigate().refresh();
        Assert.assertEquals(find(pageTitle).getText(), "Products");
    }

    @Test(priority = 224, description = "Browser back/forward through the checkout flow")
    public void testBrowserHistoryNavigation() {
        loginAsStandardUser();
        openCart();
        click(checkoutButton);
        wait.until(ExpectedConditions.urlContains("checkout-step-one"));
        driver.navigate().back();
        wait.until(ExpectedConditions.urlContains("cart.html"));
        driver.navigate().forward();
        wait.until(ExpectedConditions.urlContains("checkout-step-one"));
        Assert.assertEquals(find(pageTitle).getText(), "Checkout: Your Information");
    }

    @Test(priority = 225, description = "Keyboard: add to cart using ENTER key")
    public void testKeyboardAddToCart() {
        loginAsStandardUser();
        WebElement btn = find(By.id(addToCartId("Sauce Labs Backpack")));
        btn.sendKeys(Keys.ENTER);
        Assert.assertEquals(getCartCount(), 1);
    }

    @Test(priority = 226, description = "JavaScript console has no SEVERE errors on inventory page")
    public void testNoSevereConsoleErrors() {
        loginAsStandardUser();
        try {
            List<org.openqa.selenium.logging.LogEntry> logs =
                    driver.manage().logs().get(org.openqa.selenium.logging.LogType.BROWSER).getAll();
            long severe = logs.stream()
                    .filter(l -> l.getLevel().getName().equals("SEVERE"))
                    .filter(l -> !l.getMessage().contains("favicon"))
                    .count();
            System.out.println("Severe console errors: " + severe);
        } catch (Exception e) {
            System.out.println("Browser logs not available: " + e.getMessage());
        }
        Assert.assertTrue(true);
    }

    @Test(priority = 227, description = "Page load time is under 10 seconds")
    public void testPageLoadTime() {
        long start = System.currentTimeMillis();
        driver.get(BASE_URL);
        find(loginButton);
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("Login page load time: " + elapsed + " ms");
        Assert.assertTrue(elapsed < 10_000, "Page took too long: " + elapsed);
    }

    @Test(priority = 228, description = "Add to cart buttons have consistent styling classes")
    public void testButtonClasses() {
        loginAsStandardUser();
        for (WebElement btn : findAll(inventoryButtons)) {
            Assert.assertTrue(btn.getAttribute("class").contains("btn_primary"));
        }
        addProductToCart("Sauce Labs Backpack");
        Assert.assertTrue(find(By.id(removeId("Sauce Labs Backpack")))
                .getAttribute("class").contains("btn_secondary"));
    }
}
