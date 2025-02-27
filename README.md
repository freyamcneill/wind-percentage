# wind-percentage 
import pandas as pd
from datetime import datetime
import numpy as np
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import PolynomialFeatures
import requests
import tkinter as tk
from tkinter import ttk

# Load the data
data = pd.read_csv('eirgrid_data.csv')  # Ensure this is the correct path to your dataset

# Convert the timestamp to datetime
data['timestamp'] = pd.to_datetime(data['timestamp'])

# Filter data for the past year
one_year_ago = datetime.now() - pd.DateOffset(years=1)
data = data[data['timestamp'] >= one_year_ago]

# Add a column for the day of the week
data['day_of_week'] = data['timestamp'].dt.dayofweek

# Function to calculate average daily demand for each month
def calculate_monthly_average_demand(data):
    monthly_averages = {}
    for month in range(1, 13):
        month_data = data[data['timestamp'].dt.month == month]
        weekdays = month_data[month_data['day_of_week'] < 5]
        weekends = month_data[month_data['day_of_week'] >= 5]
        
        avg_weekday_demand = weekdays['demand'].mean()
        avg_weekend_demand = weekends['demand'].mean()
        
        monthly_averages[month] = {
            'weekday': avg_weekday_demand,
            'weekend': avg_weekend_demand
        }
    return monthly_averages

# Calculate monthly average demand
monthly_averages = calculate_monthly_average_demand(data)

# Polynomial regression model for wind energy prediction
wind_speed = np.array([7.2, 10.8, 14.4, 18, 36, 54, 72, 90]).reshape(-1, 1)  # Wind speed in km/h
energy_production_mwh = np.array([0, 1200, 2400, 3600, 28800, 81000, 144000, 180000])  # Energy over 24h in MWh

# Transform the data to include polynomial features
poly = PolynomialFeatures(degree=2)  # You can adjust the degree as needed
wind_speed_poly = poly.fit_transform(wind_speed)

# Train the polynomial regression model
model = LinearRegression()
model.fit(wind_speed_poly, energy_production_mwh)

def predict_energy_production(wind_speeds_kph):
    # Transform the input data
    wind_speeds_poly = poly.transform(np.array(wind_speeds_kph).reshape(-1, 1))
    
    # Predict energy production in MWh
    predictions = []
    for i, speed in enumerate(wind_speeds_kph):
        if speed <= 7.2 or speed > 90:
            predictions.append(0)  # No energy production outside the specified range
        else:
            predictions.append(model.predict(wind_speeds_poly[i].reshape(1, -1))[0])
    return np.array(predictions)

def fetch_wind_forecast(location):
    try:
        # Open-Meteo API call
        url = f"https://api.open-meteo.com/v1/forecast?latitude={location['lat']}&longitude={location['lon']}&hourly=wind_speed_10m&timezone=auto"
        response = requests.get(url)
        response.raise_for_status()  # Raises an HTTPError for bad responses

        data = response.json()

        # Validate the response structure
        if 'hourly' not in data or 'wind_speed_10m' not in data['hourly']:
            raise ValueError("Unexpected API response structure")

        # Extract hourly wind speeds for the next 7 days
        hourly_wind_speeds = np.array(data['hourly']['wind_speed_10m'])

        # Calculate daily average wind speeds
        daily_average_wind_speeds = hourly_wind_speeds.reshape(-1, 24).mean(axis=1)

        return daily_average_wind_speeds[:7]

    except requests.exceptions.RequestException as e:
        print(f"Network error: {e}")
        return []
    except ValueError as e:
        print(f"Data error: {e}")
        return []

def calculate_wind_energy_percentage(month, day_type, selected_day, wind_speeds_kph):
    # Get average consumption for the selected day
    avg_consumption = monthly_averages[month][day_type]
    
    # Predict energy production for the selected day
    predicted_production = predict_energy_production([wind_speeds_kph[selected_day - 1]])[0]
    
    # Calculate percentage of demand met by wind energy
    percentage = (predicted_production / avg_consumption) * 100
    return percentage

def calculate_and_display():
    try:
        month = int(month_var.get())
        day_type = day_type_var.get()
        selected_day = int(day_var.get())
        location = {'lat': 53.3498, 'lon': -6.2603}  # Example coordinates for Dublin, Ireland
        wind_speeds = fetch_wind_forecast(location)
        percentage = calculate_wind_energy_percentage(month, day_type, selected_day, wind_speeds)
        percentage_label.config(text=f"Wind Energy Contribution: {percentage:.2f}%")
    except Exception as e:
        percentage_label.config(text=f"Error: {e}")

# Set up the main application window
root = tk.Tk()
root.title("Wind Energy Calculator")
root.geometry("300x300")  # Set the window size

# Create a frame for better layout management
frame = ttk.Frame(root, padding="10")
frame.pack(fill='both', expand=True)

# Display current date
current_date = datetime.now().strftime("%Y-%m-%d")
ttk.Label(frame, text=f"Current Date: {current_date}", font=("Arial", 12)).pack(pady=5)

# Month selection
ttk.Label(frame, text="Select Month:", font=("Arial", 12)).pack(pady=5)
month_var = tk.StringVar(value=str(datetime.now().month))
month_combobox = ttk.Combobox(frame, textvariable=month_var, values=[str(i) for i in range(1, 13)], state="readonly")
month_combobox.pack()

# Day type selection
ttk.Label(frame, text="Select Day Type:", font=("Arial", 12)).pack(pady=5)
day_type_var = tk.StringVar(value='weekday')
ttk.Radiobutton(frame, text="Weekday", variable=day_type_var, value='weekday').pack()
ttk.Radiobutton(frame, text="Weekend", variable=day_type_var, value='weekend').pack()

# Day selection within the next week
ttk.Label(frame, text="Select Day (1-7):", font=("Arial", 12)).pack(pady=5)
day_var = tk.StringVar(value='1')
day_combobox = ttk.Combobox(frame, textvariable=day_var, values=[str(i) for i in range(1, 8)], state="readonly")
day_combobox.pack()

# Calculate button
calculate_button = ttk.Button(frame, text="Calculate", command=calculate_and_display)
calculate_button.pack(pady=10)

# Percentage display
percentage_label = ttk.Label(frame, text="", font=("Arial", 24, "bold"))
percentage_label.pack(pady=20)

# Run the application
root.mainloop()
