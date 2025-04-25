import streamlit as st
import json
import os

# ---------- File Setup ----------
USER_DATA_FILE = "user_data.json"
if not os.path.exists(USER_DATA_FILE):
    with open(USER_DATA_FILE, "w") as f:
        json.dump({}, f)

# ---------- Helper Functions ----------
def load_user_data():
    with open(USER_DATA_FILE, "r") as f:
        return json.load(f)

def save_user_data(data):
    with open(USER_DATA_FILE, "w") as f:
        json.dump(data, f, indent=4)

def authenticate(username, password):
    users = load_user_data()
    return username in users and users[username]["password"] == password

def register_user(username, password):
    users = load_user_data()
    if username in users:
        return False
    users[username] = {"password": password, "savings": {}}
    save_user_data(users)
    return True

def get_user_savings(username):
    return load_user_data()[username].get("savings", {})

def update_user_savings(username, savings):
    data = load_user_data()
    data[username]["savings"] = savings
    save_user_data(data)

# ---------- App State ----------
if "logged_in" not in st.session_state:
    st.session_state.logged_in = False
if "username" not in st.session_state:
    st.session_state.username = ""

# ---------- Login/Register Screen ----------
st.title("📘 School Savings Tracker")

if not st.session_state.logged_in:
    tab1, tab2 = st.tabs(["🔐 Login", "📝 Register"])

    with tab1:
        username = st.text_input("Username")
        password = st.text_input("Password", type="password")
        if st.button("Login"):
            if authenticate(username, password):
                st.session_state.logged_in = True
                st.session_state.username = username
                st.success("Logged in successfully!")
                st.experimental_rerun()
            else:
                st.error("Incorrect username or password.")

    with tab2:
        new_username = st.text_input("New Username")
        new_password = st.text_input("New Password", type="password")
        if st.button("Register"):
            if register_user(new_username, new_password):
                st.success("User registered! You can now log in.")
            else:
                st.error("Username already exists.")

else:
    username = st.session_state.username
    st.sidebar.write(f"👋 Logged in as: **{username}**")

    st.header("💸 Enter Your Monthly Info")
    income = st.number_input("Monthly Income", min_value=0.0)
    expenses = st.number_input("Monthly Expenses", min_value=0.0)
    current_savings = st.number_input("Current Total Savings", min_value=0.0)

    st.subheader("🎯 Set Savings Goals")
    goal_books = st.number_input("Books Goal", min_value=0.0)
    goal_tuition = st.number_input("Tuition Goal", min_value=0.0)
    goal_meal = st.number_input("Meal Plan Goal", min_value=0.0)

    distribution_method = st.selectbox(
        "Choose how to distribute leftover savings:",
        ["Even Split", "Fixed Percentages", "Proportional to Goal Size"]
    )

    if st.button("Calculate & Save"):
        leftover = income - expenses
        total_goals = goal_books + goal_tuition + goal_meal

        if distribution_method == "Even Split":
            books = tuition = meal = leftover / 3 if leftover > 0 else 0
        elif distribution_method == "Fixed Percentages":
            books = leftover * 0.3
            tuition = leftover * 0.5
            meal = leftover * 0.2
        else:
            books = (goal_books / total_goals) * leftover if total_goals > 0 else 0
            tuition = (goal_tuition / total_goals) * leftover if total_goals > 0 else 0
            meal = (goal_meal / total_goals) * leftover if total_goals > 0 else 0

        savings_data = {
            "income": income,
            "expenses": expenses,
            "current_savings": current_savings,
            "books_saved": books,
            "tuition_saved": tuition,
            "meal_saved": meal,
            "distribution_method": distribution_method
        }
        update_user_savings(username, savings_data)
        st.success("Savings calculated and saved!")

    # Display saved data
    user_data = get_user_savings(username)
    if user_data:
        st.subheader("📊 Savings Breakdown")
        st.write(user_data)
        st.progress(min(user_data.get("books_saved", 0) / (goal_books or 1), 1.0))
        st.progress(min(user_data.get("tuition_saved", 0) / (goal_tuition or 1), 1.0))
        st.progress(min(user_data.get("meal_saved", 0) / (goal_meal or 1), 1.0))

    if st.button("Log Out"):
        st.session_state.logged_in = False
        st.session_state.username = ""
        st.experimental_rerun()
