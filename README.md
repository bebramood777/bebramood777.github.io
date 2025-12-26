from flask import Flask, render_template, request, jsonify, session
from flask_cors import CORS
import json
import os
import random
from datetime import datetime
import pandas as pd
import numpy as np


app = Flask(__name__)
app.secret_key = 'your_secret_key_here' 
CORS(app)


users_data = {}
assistant_name = "Jason"

class PopulationAnalyzer:
  
    
    def __init__(self):
      
        self.population_data = self.load_simulated_data()
        
      
        self.holidays = {
            "2025-01-01": "New Year's Day",
            "2025-07-04": "Independence Day",
            "2025-12-25": "Christmas Day"
        }
    
    def load_simulated_data(self):
     
        
        data = {
            "states": {
                "CA": {
                    "name": "California",
                    "cities": {
                        "Los Angeles": {"population": 3976000, "density": 3241, "current_trend": "high"},
                        "San Francisco": {"population": 883000, "density": 7266, "current_trend": "high"},
                        "Death Valley": {"population": 1000, "density": 1.8, "current_trend": "low"}
                    }
                },
                "NY": {
                    "name": "New York",
                    "cities": {
                        "New York City": {"population": 8419000, "density": 10831, "current_trend": "high"},
                        "Albany": {"population": 98251, "density": 1795, "current_trend": "medium"}
                    }
                },
                "TX": {
                    "name": "Texas",
                    "cities": {
                        "Houston": {"population": 2313000, "density": 1429, "current_trend": "medium"},
                        "Marfa": {"population": 1721, "density": 1.9, "current_trend": "low"}
                    }
                }
            }
        }
        return data
    
    def check_holiday_effect(self):
       
        today = datetime.now().strftime("%Y-%m-%d")
        if today in self.holidays:
            return f"Сегодня {self.holidays[today]}. В праздники плотность населения может быть ниже."
        return None
    
    def analyze_density(self, state, city):
        """Анализ плотности населения для города"""
        try:
      
            city_data = self.population_data["states"][state.upper()]["cities"][city]
            
            
            holiday_msg = self.check_holiday_effect()
            
           
            density = city_data["density"]
            if density > 5000:
                level = "очень высокая"
                recommendation = "Избегайте центра города в часы пик"
            elif density > 2000:
                level = "высокая"
                recommendation = "Планируйте маршрут заранее"
            elif density > 100:
                level = "средняя"
                recommendation = "Комфортная плотность для посещения"
            else:
                level = "низкая"
                recommendation = "Идеально для уединенного отдыха"
            
         
            trend = city_data["current_trend"]
            trend_msg = {
                "high": f"Сейчас в {city} ожидается много людей",
                "medium": f"В {city} умеренное количество людей",
                "low": f"Сейчас в {city} мало людей"
            }.get(trend, "")
            
            return {
                "city": city,
                "state": self.population_data["states"][state.upper()]["name"],
                "population": f"{city_data['population']:,}",
                "density": f"{density} чел/км²",
                "level": level,
                "recommendation": recommendation,
                "trend": trend_msg,
                "holiday_effect": holiday_msg,
                "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            }
            
        except KeyError:
            return None


analyzer = PopulationAnalyzer()

@app.route('/')
def home():
   
    return render_template('index.html', assistant_name=assistant_name)

@app.route('/api/chat', methods=['POST'])
def chat():
   
    data = request.json
    user_message = data.get('message', '').strip().lower()
    user_id = data.get('user_id', 'default_user')
    
   
    if user_id not in users_data:
        users_data[user_id] = {
            "step": 0,
            "state": None,
            "city": None
        }
    
    session_data = users_data[user_id]
    response = {"assistant": assistant_name, "message": "", "next_step": None}
    
    
    if session_data["step"] == 0:
        response["message"] = f"Hi! Я {assistant_name}, ваш ассистент по анализу плотности населения в США. В каком штате вы находитесь? (например: CA, NY, TX)"
        session_data["step"] = 1
    
    elif session_data["step"] == 1:
        session_data["state"] = user_message.upper()
        response["message"] = f"Отлично! А какой город в штате {session_data['state']} вас интересует?"
        session_data["step"] = 2
    
    elif session_data["step"] == 2:
        session_data["city"] = user_message.title()
        
      
        result = analyzer.analyze_density(session_data["state"], session_data["city"])
        
        if result:
            response["message"] = (
                f"Анализ для {result['city']}, {result['state']}:\n"
                f"• Население: {result['population']} человек\n"
                f"• Плотность: {result['density']}\n"
                f"• Уровень: {result['level']}\n"
                f"• Тренд: {result['trend']}\n"
                f"• Рекомендация: {result['recommendation']}\n"
            )
            
            if result["holiday_effect"]:
                response["message"] += f"\n⚠️ {result['holiday_effect']}"
        else:
            response["message"] = "Извините, не нашел данных для этого города. Попробуйте другой город."
        
        # Сброс диалога
        session_data["step"] = 0
        response["next_step"] = "restart"
    
    return jsonify(response)

@app.route('/api/map_data')
def get_map_data():
   
    markers = [
        {"lat": 34.0522, "lng": -118.2437, "name": "Los Angeles", "density": "high"},
        {"lat": 40.7128, "lng": -74.0060, "name": "New York City", "density": "very high"},
        {"lat": 36.7378, "lng": -118.1451, "name": "Death Valley", "density": "very low"},
        {"lat": 30.3099, "lng": -103.2402, "name": "Marfa, TX", "density": "low"}
    ]
    
    return jsonify({
        "center": {"lat": 39.8283, "lng": -98.5795},  # Центр США
        "zoom": 4,
        "markers": markers
    })

@app.route('/api/low_density_locations')
def get_low_density_locations():
    today = datetime.now().strftime("%Y-%m-%d")
    
    locations = [
        {
            "name": "Death Valley, CA",
            "lat": 36.7378,
            "lng": -118.1451,
            "reason": "Национальный парк, малонаселенная пустыня",
            "density": "1.8 чел/км²"
        },
        {
            "name": "Marfa, TX",
            "lat": 30.3099,
            "lng": -103.2402,
            "reason": "Маленький город в пустыне",
            "density": "1.9 чел/км²"
        },
        {
            "name": "Yellowstone National Park",
            "lat": 44.4280,
            "lng": -110.5885,
            "reason": "Национальный парк, природа",
            "density": "2.1 чел/км²"
        }
    ]
    
   
    if today in analyzer.holidays:
        locations.append({
            "name": "❗ ВНИМАНИЕ",
            "lat": 39.8283,
            "lng": -98.5795,
            "reason": f"Сегодня {analyzer.holidays[today]}. Даже в городах может быть меньше людей.",
            "density": "Праздничный эффект"
        })
    
    return jsonify({"locations": locations, "date": today})

if __name__ == '__main__':
    app.run(debug=True, port=5000)
