import json
import numpy as np
import pandas as pd
import scipy.stats as stats

try:
    import requests
except ModuleNotFoundError:
    import micropip
    micropip.install("requests")
    import requests

# API Endpoints
CURRENT_QUIZ_URL = "https://api.jsonserve.com/rJvd7g"
HISTORICAL_QUIZ_URL = "https://api.jsonserve.com/XgAgFJ"

# Fetch Data with Error Handling
def fetch_data(url):
    try:
        response = requests.get(url, timeout=5)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data from {url}: {e}")
        return None

# Load Data
current_quiz_data = fetch_data(CURRENT_QUIZ_URL) or {}
historical_quiz_data = fetch_data(HISTORICAL_QUIZ_URL) or []

# Analyze Performance
def analyze_performance(current, history):
    performance = {}
    
    for quiz in history:
        if not isinstance(quiz, dict) or 'response_map' not in quiz or 'correct_answers' not in quiz:
            continue
        
        correct_answers = quiz['correct_answers'] if isinstance(quiz['correct_answers'], dict) else {}
        
        for q_id, selected_option in quiz['response_map'].items():
            if q_id not in performance:
                performance[q_id] = {'attempts': 0, 'correct': 0}
            performance[q_id]['attempts'] += 1
            if correct_answers.get(q_id) == selected_option:
                performance[q_id]['correct'] += 1
    
    return performance

# Generate Insights
def generate_insights(performance):
    insights = []
    for q_id, stats in performance.items():
        accuracy = stats['correct'] / stats['attempts'] if stats['attempts'] > 0 else 0
        if accuracy < 0.5:
            insights.append(f"Focus more on question {q_id} - Low accuracy: {accuracy:.2f}")
    return insights

# Generate Recommendations
def generate_recommendations(insights):
    recommendations = []
    for insight in insights:
        recommendations.append(f"Revise concepts related to {insight.split('-')[0].strip()}")
    return recommendations

# Define Student Persona
def define_student_persona(performance):
    strengths = []
    weaknesses = []
    
    for q_id, stats in performance.items():
        accuracy = stats['correct'] / stats['attempts'] if stats['attempts'] > 0 else 0
        if accuracy > 0.8:
            strengths.append(q_id)
        elif accuracy < 0.5:
            weaknesses.append(q_id)
    
    persona = {
        "Strong Performer" if len(strengths) > len(weaknesses) else "Needs Improvement": {
            "Strengths": strengths,
            "Weaknesses": weaknesses
        }
    }
    
    return persona

# Predict NEET Rank
def predict_neet_rank(performance):
    total_questions = sum(stats['attempts'] for stats in performance.values())
    correct_answers = sum(stats['correct'] for stats in performance.values())
    accuracy = correct_answers / total_questions if total_questions > 0 else 0
    
    mean_neet_score = 600  # Hypothetical average NEET score
    std_dev = 50  # Hypothetical standard deviation
    predicted_score = mean_neet_score * accuracy
    
    rank_estimate = stats.norm.sf(predicted_score, mean_neet_score, std_dev) * 1000000  # Approximate rank
    
    return max(1, int(rank_estimate))

# Execute Analysis
if current_quiz_data and historical_quiz_data:
    performance = analyze_performance(current_quiz_data, historical_quiz_data)
    insights = generate_insights(performance)
    recommendations = generate_recommendations(insights)
    persona = define_student_persona(performance)
    predicted_rank = predict_neet_rank(performance)
    
    # Display Results
    print("Insights:")
    print("\n".join(insights) if insights else "No insights available.")
    print("\nRecommendations:")
    print("\n".join(recommendations) if recommendations else "No recommendations available.")
    print("\nStudent Persona:")
    print(json.dumps(persona, indent=4))
    print("\nPredicted NEET Rank:", predicted_rank)
else:
    print("Data could not be loaded. Please check API connectivity.")
