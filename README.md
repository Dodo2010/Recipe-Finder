import sys
import requests
import json
import os
from pathlib import Path
from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QLabel, QVBoxLayout, QHBoxLayout,
    QLineEdit, QPushButton, QTextEdit, QComboBox, QListWidget, 
    QAction, QFileDialog, QWidget, QMessageBox, QInputDialog,
    QTabWidget, QScrollArea, QFrame, QGridLayout, QGroupBox, QToolBar, QDialog, QMenuBar, QMenu
)
from PyQt5.QtCore import Qt
from PyQt5.QtGui import QFont, QIcon




# Spoonacular API Key
API_KEY = "107b8aad651749a0be6b4a39265c7019"
BASE_URL = "https://api.spoonacular.com/recipes/"


def search_recipes(query, number=5, cuisine=None):
    """Search for recipes using Spoonacular API."""
    url = f"{BASE_URL}complexSearch"
    params = {"apiKey": API_KEY, "query": query, "number": number}
    if cuisine:
        params["cuisine"] = cuisine
    response = requests.get(url, params=params)
    return response.json().get("results", []) if response.status_code == 200 else []


def get_recipe_info(recipe_id):
    """Get detailed recipe information."""
    url = f"{BASE_URL}{recipe_id}/information"
    params = {"apiKey": API_KEY}
    response = requests.get(url, params=params)
    return response.json() if response.status_code == 200 else {}


def get_nutritional_info(recipe_id):
    """Fetch nutritional information for a recipe."""
    url = f"{BASE_URL}{recipe_id}/nutritionWidget.json"
    params = {"apiKey": API_KEY}
    response = requests.get(url, params=params)
    return response.json() if response.status_code == 200 else {}


class RecipeFinderApp(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Recipe Finder Pro")
        self.setGeometry(100, 100, 1200, 800)
        
        # Initialize data storage
        self.data_dir = Path.home() / '.recipe_finder'
        self.recipes_file = self.data_dir / 'saved_recipes.json'
        self.meal_plans_file = self.data_dir / 'meal_plans.json'
        
        # Create data directory if it doesn't exist
        self.data_dir.mkdir(exist_ok=True)
        
        # Load saved data
        self.saved_recipes = self.load_saved_recipes()
        self.current_meal_plan = None
        
        # Initialize meal plans storage
        self.meal_plans = self.load_meal_plans()
        
        # Create main tab widget
        self.tabs = QTabWidget()
        self.setCentralWidget(self.tabs)
        
        # Initialize tabs
        self.setup_search_tab()
        self.setup_saved_recipes_tab()
        self.setup_meal_planner_tab()
        self.setup_shopping_tab()
        
        # Style
        self.setStyleSheet("""
            QMainWindow {
                background-color: #f0f0f0;
            }
            QTabWidget::pane {
                border: 1px solid #cccccc;
                background: white;
            }
            QTabBar::tab {
                background: #e1e1e1;
                padding: 10px 20px;
                margin: 2px;
            }
            QTabBar::tab:selected {
                background: #ffffff;
                border-bottom: 2px solid #2196F3;
            }
            QPushButton {
                background-color: #2196F3;
                color: white;
                padding: 8px 15px;
                border: none;
                border-radius: 4px;
            }
            QPushButton:hover {
                background-color: #1976D2;
            }
            QLineEdit {
                padding: 8px;
                border: 1px solid #cccccc;
                border-radius: 4px;
            }
            QListWidget {
                border: 1px solid #cccccc;
                border-radius: 4px;
            }
        """)
        
        # Add theme toggle button to the main window
        self.theme_button = QPushButton("Toggle Theme")
        self.theme_button.clicked.connect(lambda: self.toggle_theme())
        self.toolbar = QToolBar("Tools")
        self.addToolBar(self.toolbar)
        self.toolbar.addWidget(self.theme_button)
        
        self.is_dark_theme = False
        self.apply_theme()
        
        # Create menu bar
        self.setup_menu_bar()

    def setup_menu_bar(self):
        """Setup the application menu bar."""
        menubar = QMenuBar(self)
        self.setMenuBar(menubar)
        
        # File menu
        file_menu = QMenu('File', self)
        menubar.addMenu(file_menu)
        exit_action = QAction('Exit', self)
        exit_action.setShortcut('Ctrl+Q')
        exit_action.triggered.connect(lambda: self.close())
        file_menu.addAction(exit_action)
        
        # View menu for theme
        view_menu = QMenu('View', self)
        menubar.addMenu(view_menu)  # Add view menu to menubar
        self.theme_action = QAction('Dark Mode', self)
        self.theme_action.setCheckable(True)
        self.theme_action.setChecked(self.is_dark_theme)
        self.theme_action.triggered.connect(self.toggle_theme)
        self.theme_action.setShortcut('Ctrl+T')
        view_menu.addAction(self.theme_action)
        
        # Help menu
        help_menu = menubar.addMenu('Help')
        about_action = QAction('About', self)
        about_action.triggered.connect(self.show_about)
        help_menu.addAction(about_action)

    def show_about(self):
        """Show the about dialog."""
        about_text = """
        <h2>Recipe Finder Pro</h2>
        <p>This app was made by Abdullah Abdullah and Yousif Al Kasab</p>
        <p>Version 3.1.0</p>
        """
        QMessageBox.about(self, "About Recipe Finder Pro", about_text)

    def setup_search_tab(self):
        search_tab = QWidget()
        layout = QHBoxLayout()
        
        # Left panel for search and results
        left_panel = QWidget()
        left_layout = QVBoxLayout()
        
        # Search group
        search_group = QGroupBox("Search Recipes")
        search_layout = QVBoxLayout()
        
        # Search controls
        search_bar = QHBoxLayout()
        self.query_input = QLineEdit()
        self.query_input.setPlaceholderText("Enter ingredients or recipe name")
        self.recipe_count = QComboBox()
        self.recipe_count.addItems(["5", "10", "15", "20"])
        self.search_button = QPushButton("Search")
        
        search_bar.addWidget(self.query_input)
        search_bar.addWidget(self.recipe_count)
        search_bar.addWidget(self.search_button)
        
        # Results list
        self.results_list = QListWidget()
        
        search_layout.addLayout(search_bar)
        search_layout.addWidget(self.results_list)
        search_group.setLayout(search_layout)
        left_layout.addWidget(search_group)
        left_panel.setLayout(left_layout)
        
        # Right panel for recipe details
        right_panel = QWidget()
        right_layout = QVBoxLayout()
        
        # Details group
        details_group = QGroupBox("Recipe Details")
        details_layout = QVBoxLayout()
        
        self.details_text = QTextEdit()
        self.details_text.setReadOnly(True)
        
        # Action buttons
        button_layout = QHBoxLayout()
        self.save_button = QPushButton("Save Recipe")
        self.nutrition_button = QPushButton("Nutrition Info")
        button_layout.addWidget(self.save_button)
        button_layout.addWidget(self.nutrition_button)
        
        details_layout.addWidget(self.details_text)
        details_layout.addLayout(button_layout)
        details_group.setLayout(details_layout)
        right_layout.addWidget(details_group)
        right_panel.setLayout(right_layout)
        
        # Add panels to main layout
        layout.addWidget(left_panel, 1)
        layout.addWidget(right_panel, 2)
        search_tab.setLayout(layout)
        
        # Connect signals
        self.search_button.clicked.connect(self.search_recipes)
        self.results_list.itemDoubleClicked.connect(self.show_recipe_details)
        self.save_button.clicked.connect(self.save_recipe)
        self.nutrition_button.clicked.connect(self.show_nutritional_info)
        
        self.tabs.addTab(search_tab, "Search")

    def setup_saved_recipes_tab(self):
        saved_tab = QWidget()
        layout = QVBoxLayout()
        
        # Add refresh button and delete button
        button_layout = QHBoxLayout()
        refresh_button = QPushButton("Refresh Saved Recipes")
        delete_button = QPushButton("Delete Selected Recipe")
        
        refresh_button.clicked.connect(self.refresh_saved_recipes)
        delete_button.clicked.connect(self.delete_selected_recipe)
        
        button_layout.addWidget(refresh_button)
        button_layout.addWidget(delete_button)
        
        self.saved_list = QListWidget()
        layout.addWidget(QLabel("Your Saved Recipes"))
        layout.addLayout(button_layout)
        layout.addWidget(self.saved_list)
        
        # Populate the list with saved recipes
        self.refresh_saved_recipes()
        
        saved_tab.setLayout(layout)
        self.tabs.addTab(saved_tab, "Saved Recipes")

    def setup_meal_planner_tab(self):
        planner_tab = QWidget()
        layout = QVBoxLayout()
        
        # Calendar widget or date selection
        date_layout = QHBoxLayout()
        self.date_input = QLineEdit()
        self.date_input.setPlaceholderText("Enter date (YYYY-MM-DD)")
        date_layout.addWidget(QLabel("Date:"))
        date_layout.addWidget(self.date_input)
        
        # Meal selection grid
        meal_grid = QGridLayout()
        self.meal_combos = []
        
        for i, meal in enumerate(["Breakfast", "Lunch", "Dinner"]):
            meal_grid.addWidget(QLabel(meal), i, 0)
            combo = QComboBox()
            combo.addItems([''] + list(self.saved_recipes.keys()))
            self.meal_combos.append(combo)
            meal_grid.addWidget(combo, i, 1)
        
        # Add Optional Juice and Dessert inputs
        self.juice_input = QLineEdit()
        self.juice_input.setPlaceholderText("Enter juice (optional)")
        self.dessert_input = QLineEdit()
        self.dessert_input.setPlaceholderText("Enter dessert (optional)")
        
        meal_grid.addWidget(QLabel("Juice (Optional):"), 3, 0)
        meal_grid.addWidget(self.juice_input, 3, 1)
        meal_grid.addWidget(QLabel("Dessert (Optional):"), 4, 0)
        meal_grid.addWidget(self.dessert_input, 4, 1)
        
        # Buttons
        button_layout = QHBoxLayout()
        self.create_plan_button = QPushButton("Create Plan")
        self.export_plan_button = QPushButton("Export Plan")
        button_layout.addWidget(self.create_plan_button)
        button_layout.addWidget(self.export_plan_button)
        
        # Add all layouts to main layout
        layout.addLayout(date_layout)
        layout.addLayout(meal_grid)
        layout.addLayout(button_layout)
        layout.addStretch()
        
        # Connect signals
        self.create_plan_button.clicked.connect(self.create_meal_plan)
        self.export_plan_button.clicked.connect(self.export_meal_plan)
        
        planner_tab.setLayout(layout)
        self.tabs.addTab(planner_tab, "Meal Planner")

    def setup_shopping_tab(self):
        """Setup the shopping list tab."""
        shopping_tab = QWidget()
        layout = QVBoxLayout()
        
        # Shopping list display
        self.shopping_list_display = QTextEdit()
        self.shopping_list_display.setReadOnly(True)
        self.shopping_list_display.setPlaceholderText("Your shopping list will appear here...")
        
        # Buttons
        button_layout = QHBoxLayout()
        generate_button = QPushButton("Generate Shopping List")
        export_button = QPushButton("Export List")
        clear_button = QPushButton("Clear List")
        
        button_layout.addWidget(generate_button)
        button_layout.addWidget(export_button)
        button_layout.addWidget(clear_button)
        
        # Connect signals
        generate_button.clicked.connect(self.generate_shopping_list)
        export_button.clicked.connect(self.export_shopping_list)
        clear_button.clicked.connect(self.clear_shopping_list)
        
        layout.addWidget(QLabel("Shopping List"))
        layout.addWidget(self.shopping_list_display)
        layout.addLayout(button_layout)
        
        shopping_tab.setLayout(layout)
        self.tabs.addTab(shopping_tab, "Shopping List")

    def search_recipes(self):
        query = self.query_input.text().strip()
        if not query:
            return
        number = int(self.recipe_count.currentText())
        recipes = search_recipes(query, number)
        self.results_list.clear()
        self.recipes = {recipe["id"]: recipe["title"] for recipe in recipes}
        for recipe in recipes:
            self.results_list.addItem(recipe["title"])

    def show_recipe_details(self, item):
        recipe_id = next((id for id, title in self.recipes.items() if title == item.text()), None)
        if recipe_id:
            recipe_info = get_recipe_info(recipe_id)
            details = f"Title: {recipe_info.get('title', 'N/A')}\n"
            details += f"Ready in: {recipe_info.get('readyInMinutes', 'N/A')} minutes\n"
            details += f"Servings: {recipe_info.get('servings', 'N/A')}\n\n"
            details += f"Ingredients:\n"
            details += "\n".join(
                [f"- {ingredient['original']}" for ingredient in recipe_info.get("extendedIngredients", [])]
            )
            details += f"\n\nInstructions:\n{recipe_info.get('instructions', 'No instructions provided.')}\n"
            self.details_text.setPlainText(details)
            self.current_recipe = recipe_info

    def save_recipe(self):
        """Save a recipe and persist to file."""
        if hasattr(self, "current_recipe") and self.current_recipe:
            self.saved_recipes[self.current_recipe["title"]] = self.current_recipe
            self.save_recipes_to_file()  # Save to file immediately
            self.refresh_saved_recipes()  # This will also refresh meal combos
            QMessageBox.information(self, "Saved", f"Recipe '{self.current_recipe['title']}' saved!")

    def show_nutritional_info(self):
        if hasattr(self, "current_recipe") and self.current_recipe:
            recipe_id = self.current_recipe["id"]
            nutrition = get_nutritional_info(recipe_id)
            if nutrition:
                info = f"Calories: {nutrition.get('calories', 'N/A')}\n"
                info += f"Protein: {nutrition.get('protein', 'N/A')}\n"
                info += f"Fat: {nutrition.get('fat', 'N/A')}\n"
                QMessageBox.information(self, "Nutritional Info", info)
            else:
                QMessageBox.warning(self, "Error", "Unable to fetch nutritional info.")

    def refresh_saved_recipes(self):
        """Update the saved recipes list widget and meal combos."""
        # Update saved recipes list
        self.saved_list.clear()
        for title in self.saved_recipes.keys():
            self.saved_list.addItem(title)
        
        # Update meal planner combos if they exist
        if hasattr(self, 'meal_combos'):
            self.refresh_meal_combos()

    def refresh_meal_combos(self):
        """Update meal combo boxes with saved recipes."""
        if hasattr(self, 'meal_combos'):
            for combo in self.meal_combos:
                current_text = combo.currentText()
                combo.clear()
                combo.addItem("Select a meal...")  # Add default option
                combo.addItems(list(self.saved_recipes.keys()))
                if current_text in self.saved_recipes:
                    combo.setCurrentText(current_text)

    def create_meal_plan(self):
        """Create a meal plan with validation."""
        if not self.saved_recipes:
            QMessageBox.warning(self, "No Recipes", "Please save some recipes first!")
            return

        date = self.date_input.text().strip()
        if not date:
            QMessageBox.warning(self, "No Date", "Please enter a date for the meal plan!")
            return

        meals = [combo.currentText() for combo in self.meal_combos]
        juice = self.juice_input.text().strip()
        dessert = self.dessert_input.text().strip()

        self.current_meal_plan = {
            "date": date,
            "meals": meals,
            "juice": juice,
            "dessert": dessert
        }
        
        # Save the meal plan
        self.meal_plans[date] = self.current_meal_plan
        self.save_meal_plans_to_file()

        QMessageBox.information(self, "Success", "Meal plan has been created!")

    def export_meal_plan(self):
        """Export the current meal plan to a text file."""
        if not hasattr(self, 'current_meal_plan') or not self.current_meal_plan:
            QMessageBox.warning(self, "No Meal Plan", "Please create a meal plan first!")
            return

        options = QFileDialog.Options()
        file_name, _ = QFileDialog.getSaveFileName(
            self,
            "Save Meal Plan",
            "",
            "Text Files (*.txt);;All Files (*)",
            options=options
        )
        
        if file_name:
            try:
                with open(file_name, 'w') as f:
                    f.write(f"**{self.current_meal_plan['date']}**\n\n")
                    f.write(f"The First Meal: {self.current_meal_plan['meals'][0]}\n")
                    f.write(f"The Second Meal: {self.current_meal_plan['meals'][1]}\n")
                    f.write(f"The Third Meal: {self.current_meal_plan['meals'][2]}\n")
                    
                    if self.current_meal_plan['juice']:
                        f.write(f"\nJuice: {self.current_meal_plan['juice']}\n")
                    if self.current_meal_plan['dessert']:
                        f.write(f"Dessert: {self.current_meal_plan['dessert']}\n")
                
                QMessageBox.information(self, "Success", "Meal plan has been exported!")
            except Exception as e:
                QMessageBox.critical(self, "Error", f"Failed to export meal plan: {str(e)}")

    def toggle_theme(self):
        """Toggle between light and dark theme."""
        self.is_dark_theme = not self.is_dark_theme
        self.theme_action.setChecked(self.is_dark_theme)
        self.apply_theme()

    def apply_theme(self):
        """Apply the current theme."""
        if self.is_dark_theme:
            self.setStyleSheet("""
                QMainWindow, QWidget {
                    background-color: #2b2b2b;
                    color: #ffffff;
                }
                QTabWidget::pane {
                    border: 1px solid #555555;
                    background: #2b2b2b;
                }
                QTabBar::tab {
                    background: #3b3b3b;
                    color: #ffffff;
                    padding: 10px 20px;
                    margin: 2px;
                }
                QTabBar::tab:selected {
                    background: #4b4b4b;
                    border-bottom: 2px solid #2196F3;
                }
                QPushButton {
                    background-color: #2196F3;
                    color: white;
                    padding: 8px 15px;
                    border: none;
                    border-radius: 4px;
                }
                QPushButton:hover {
                    background-color: #1976D2;
                }
                QLineEdit, QTextEdit, QListWidget, QComboBox {
                    background-color: #3b3b3b;
                    color: #ffffff;
                    padding: 8px;
                    border: 1px solid #555555;
                    border-radius: 4px;
                }
                QComboBox QAbstractItemView {
                    background-color: #3b3b3b;
                    color: #ffffff;
                    selection-background-color: #2196F3;
                }
                QMenuBar {
                    background-color: #2b2b2b;
                    color: #ffffff;
                }
                QMenuBar::item {
                    background-color: #2b2b2b;
                    color: #ffffff;
                }
                QMenuBar::item:selected {
                    background-color: #3b3b3b;
                }
                QMenu {
                    background-color: #2b2b2b;
                    color: #ffffff;
                }
                QMenu::item:selected {
                    background-color: #3b3b3b;
                }
                QGroupBox {
                    color: #ffffff;
                    border: 1px solid #555555;
                }
                QScrollBar:vertical {
                    background-color: #2b2b2b;
                    width: 12px;
                }
                QScrollBar::handle:vertical {
                    background-color: #555555;
                    border-radius: 6px;
                }
                QScrollBar::add-line:vertical, QScrollBar::sub-line:vertical {
                    background: none;
                }
            """)
        else:
            self.setStyleSheet("""
                QMainWindow, QWidget {
                    background-color: #f0f0f0;
                    color: #000000;
                }
                QTabWidget::pane {
                    border: 1px solid #cccccc;
                    background: white;
                }
                QTabBar::tab {
                    background: #e1e1e1;
                    padding: 10px 20px;
                    margin: 2px;
                }
                QTabBar::tab:selected {
                    background: #ffffff;
                    border-bottom: 2px solid #2196F3;
                }
                QPushButton {
                    background-color: #2196F3;
                    color: white;
                    padding: 8px 15px;
                    border: none;
                    border-radius: 4px;
                }
                QPushButton:hover {
                    background-color: #1976D2;
                }
                QLineEdit, QTextEdit, QListWidget, QComboBox {
                    background-color: #ffffff;
                    color: #000000;
                    padding: 8px;
                    border: 1px solid #cccccc;
                    border-radius: 4px;
                }
                QComboBox QAbstractItemView {
                    background-color: #ffffff;
                    color: #000000;
                    selection-background-color: #2196F3;
                }
                QMenuBar {
                    background-color: #f0f0f0;
                }
                QMenuBar::item:selected {
                    background-color: #e1e1e1;
                }
                QMenu::item:selected {
                    background-color: #e1e1e1;
                }
                QGroupBox {
                    border: 1px solid #cccccc;
                }
                QScrollBar:vertical {
                    background-color: #f0f0f0;
                    width: 12px;
                }
                QScrollBar::handle:vertical {
                    background-color: #cccccc;
                    border-radius: 6px;
                }
                QScrollBar::add-line:vertical, QScrollBar::sub-line:vertical {
                    background: none;
                }
            """)

    def load_saved_recipes(self):
        """Load saved recipes from JSON file."""
        try:
            if self.recipes_file.exists():
                with open(self.recipes_file, 'r') as f:
                    return json.load(f)
        except Exception as e:
            QMessageBox.warning(self, "Load Error", f"Could not load saved recipes: {str(e)}")
        return {}

    def save_recipes_to_file(self):
        """Save recipes to JSON file."""
        try:
            with open(self.recipes_file, 'w') as f:
                json.dump(self.saved_recipes, f)
        except Exception as e:
            QMessageBox.critical(self, "Save Error", f"Could not save recipes: {str(e)}")

    def delete_recipe(self, recipe_title):
        """Delete a recipe and update file."""
        if recipe_title in self.saved_recipes:
            del self.saved_recipes[recipe_title]
            self.save_recipes_to_file()
            self.refresh_saved_recipes()
            self.refresh_meal_combos()
            QMessageBox.information(self, "Deleted", f"Recipe '{recipe_title}' deleted!")

    def delete_selected_recipe(self):
        """Delete the selected recipe from the saved recipes list."""
        current_item = self.saved_list.currentItem()
        if current_item is None:
            QMessageBox.warning(self, "No Selection", "Please select a recipe to delete!")
            return
            
        recipe_title = current_item.text()
        reply = QMessageBox.question(self, 'Confirm Delete',
                                   f'Are you sure you want to delete "{recipe_title}"?',
                                   QMessageBox.Yes | QMessageBox.No, QMessageBox.No)
        
        if reply == QMessageBox.Yes:
            self.delete_recipe(recipe_title)

    def closeEvent(self, event):
        """Save all data when closing the application."""
        try:
            self.save_recipes_to_file()
            self.save_meal_plans_to_file()
            event.accept()
        except Exception as e:
            reply = QMessageBox.question(self, 'Save Error',
                                       f'Error saving data: {str(e)}\nDo you still want to quit?',
                                       QMessageBox.Yes | QMessageBox.No, QMessageBox.No)
            
            if reply == QMessageBox.Yes:
                event.accept()
            else:
                event.ignore()

    def load_meal_plans(self):
        """Load meal plans from JSON file."""
        try:
            if self.meal_plans_file.exists():
                with open(self.meal_plans_file, 'r') as f:
                    return json.load(f)
        except Exception as e:
            QMessageBox.warning(self, "Load Error", f"Could not load meal plans: {str(e)}")
        return {}

    def save_meal_plans_to_file(self):
        """Save meal plans to JSON file."""
        try:
            with open(self.meal_plans_file, 'w') as f:
                json.dump(self.meal_plans, f)
        except Exception as e:
            QMessageBox.critical(self, "Save Error", f"Could not save meal plans: {str(e)}")

    def generate_shopping_list(self):
        """Generate a shopping list for a selected meal plan."""
        if not self.meal_plans:
            QMessageBox.warning(self, "No Meal Plans", "Please create a meal plan first!")
            return

        # Create a dialog for meal plan selection
        dialog = QDialog(self)
        dialog.setWindowTitle("Select Meal Plan")
        dialog.setModal(True)
        layout = QVBoxLayout()

        # Add label
        layout.addWidget(QLabel("Select a meal plan:"))

        # Add list widget with meal plans
        list_widget = QListWidget()
        for date in sorted(self.meal_plans.keys()):
            list_widget.addItem(date)
        layout.addWidget(list_widget)

        # Add buttons
        button_box = QHBoxLayout()
        select_button = QPushButton("Generate List")
        cancel_button = QPushButton("Cancel")

        button_box.addWidget(select_button)
        button_box.addWidget(cancel_button)
        layout.addLayout(button_box)

        dialog.setLayout(layout)

        # Connect buttons
        select_button.clicked.connect(dialog.accept)
        cancel_button.clicked.connect(dialog.reject)

        # Show dialog
        if dialog.exec_() == QDialog.Accepted and list_widget.currentItem():
            selected_date = list_widget.currentItem().text()
            selected_plan = self.meal_plans[selected_date]
            self.generate_shopping_list_for_plan(selected_plan)

    def generate_shopping_list_for_plan(self, meal_plan):
        """Generate a shopping list for a specific meal plan."""
        shopping_items = {}  # Dictionary to store ingredients and their quantities
        
        # Format the shopping list
        formatted_list = []
        formatted_list.append(f"Shopping List for {meal_plan['date']}\n")
        formatted_list.append("=" * 40 + "\n\n")
        
        # Add meals section
        formatted_list.append("Planned Meals:")
        formatted_list.append("-" * 20)
        meal_types = ["Breakfast", "Lunch", "Dinner"]
        for meal_type, meal_name in zip(meal_types, meal_plan['meals']):
            if meal_name and meal_name != "Select a meal...":
                formatted_list.append(f"{meal_type}: {meal_name}")
        
        # Add optional items if they exist
        if meal_plan.get('juice'):
            formatted_list.append(f"Juice: {meal_plan['juice']}")
        if meal_plan.get('dessert'):
            formatted_list.append(f"Dessert: {meal_plan['dessert']}")
        
        formatted_list.append("\n" + "=" * 40 + "\n")
        formatted_list.append("Required Ingredients:")
        formatted_list.append("-" * 20 + "\n")
        
        # Process each meal in the plan
        for meal_name in meal_plan['meals']:
            if meal_name and meal_name != "Select a meal..." and meal_name in self.saved_recipes:
                recipe = self.saved_recipes[meal_name]
                if 'extendedIngredients' in recipe:
                    for ingredient in recipe['extendedIngredients']:
                        name = ingredient.get('name', '').lower()
                        amount = ingredient.get('amount', 0)
                        unit = ingredient.get('unit', '')
                        
                        if name:
                            if name in shopping_items:
                                if shopping_items[name]['unit'] == unit:
                                    shopping_items[name]['amount'] += amount
                            else:
                                shopping_items[name] = {
                                    'amount': amount,
                                    'unit': unit
                                }

        if not shopping_items:
            QMessageBox.warning(self, "No Ingredients", "No ingredients found in the meal plan!")
            return
        
        # Sort ingredients alphabetically
        for name in sorted(shopping_items.keys()):
            item = shopping_items[name]
            amount = item['amount']
            unit = item['unit']
            
            # Format the line item
            if unit:
                formatted_list.append(f"- {name.title()}: {amount:.2f} {unit}")
            else:
                formatted_list.append(f"- {name.title()}: {amount:.2f}")

        # Display in the text edit
        self.shopping_list_display.setPlainText("\n".join(formatted_list))
        QMessageBox.information(self, "Success", "Shopping list has been generated!")

    def export_shopping_list(self):
        """Export the current shopping list to a file."""
        if self.shopping_list_display.toPlainText().strip() == "":
            QMessageBox.warning(self, "Empty List", "Please generate a shopping list first!")
            return

        options = QFileDialog.Options()
        file_name, _ = QFileDialog.getSaveFileName(
            self,
            "Save Shopping List",
            "",
            "Text Files (*.txt);;All Files (*)",
            options=options
        )
        
        if file_name:
            try:
                with open(file_name, 'w') as f:
                    f.write(self.shopping_list_display.toPlainText())
                QMessageBox.information(self, "Success", "Shopping list has been exported!")
            except Exception as e:
                QMessageBox.critical(self, "Error", f"Failed to save shopping list: {str(e)}")

    def clear_shopping_list(self):
        """Clear the current shopping list."""
        self.shopping_list_display.clear()

if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = RecipeFinderApp()
    window.show()
    sys.exit(app.exec_())
