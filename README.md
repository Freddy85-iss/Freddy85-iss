from telegram.ext import Updater, CommandHandler, MessageHandler, Filters
import requests
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestRegressor
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error
import logging
import time

# === Configuration ===
API_TOKEN = "7943481895:AAEC_uGT4HKgjN1fTLGKXtpHnsvEYro5w20

For a description of the Bot API, see this page: https://core.telegram.org/bots/api"
API_FOOTBALL_KEY = "VOTRE_CLE_API_FOOTBALL"
FOOTBALL_ENDPOINT = "https://v3.football.api-sports.io"
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

# === Modèle d'IA ===
def train_model():
    """Entraîne un modèle pour prédire les scores."""
    try:
        # Charger les données historiques
        data = pd.read_csv("historical_matches.csv")  # Fichier CSV contenant les données
        data.fillna(0, inplace=True)  # Gérer les valeurs manquantes

        # Ajouter des caractéristiques supplémentaires
        data["Goal_Difference"] = abs(data["Home_Goals"] - data["Away_Goals"])
        data["Total_Goals"] = data["Home_Goals"] + data["Away_Goals"]

        # Séparer les variables explicatives et cibles
        X = data[["Home_Goals", "Away_Goals", "Goal_Difference", "Total_Goals"]]
        y = data[["Home_Final_Score", "Away_Final_Score"]]

        # Diviser les données en ensembles d'entraînement et de test
        X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

        # Entraîner un modèle de Random Forest
        model = RandomForestRegressor(n_estimators=100, random_state=42)
        model.fit(X_train, y_train)

        # Évaluer le modèle
        predictions = model.predict(X_test)
        mse = mean_squared_error(y_test, predictions)
        logging.info(f"Erreur quadratique moyenne du modèle : {mse}")

        return model
    except Exception as e:
        logging.error(f"Erreur lors de l'entraînement du modèle : {e}")
        return None

model = train_model()

# === Récupération des données en temps réel ===
def fetch_live_data():
    """Récupère les données des matchs en direct."""
    try:
        headers = {"x-rapidapi-key": API_FOOTBALL_KEY}
        live_url = f"{FOOTBALL_ENDPOINT}/fixtures?live=all"
        response = requests.get(live_url, headers=headers, timeout=10)
        response.raise_for_status()

        matches = response.json().get("response", [])
        live_data = []
        for match in matches:
            fixture = match.get("fixture", {})
            teams = match.get("teams", {})
            goals = match.get("goals", {})
            events = match.get("events", [])

            # Collecter les moments des buts
            goal_times = [
                {"Team": event["team"]["name"], "Minute": event["time"]["elapsed"]}
                for event in events if event["type"] == "Goal"
            ]

            live_data.append({
                "Home_Team": teams.get("home", {}).get("name"),
                "Away_Team": teams.get("away", {}).get("name"),
                "Home_Goals": goals.get("home"),
                "Away_Goals": goals.get("away"),
                "Elapsed_Time": fixture.get("status", {}).get("elapsed"),
                "Goal_Times": goal_times,
            })
        return pd.DataFrame(live_data)
    except Exception as e:
        logging.error(f"Erreur lors de la collecte des données en direct : {e}")
        return pd.DataFrame()

# === Prédiction des scores ===
def predict_scores(home_team, away_team):
    """Prédit les scores d'un match donné."""
    try:
        live_data = fetch_live_data()

        # Filtrer les données pour le match spécifié
        match_data = live_data[(live_data["Home_Team"] == home_team) & (live_data["Away_Team"] == away_team)]
        if match_data.empty:
            return "Aucune donnée trouvée pour ce match."

        # Préparer les données pour la prédiction
        features = match_data[["Home_Goals", "Away_Goals"]]
        predictions = model.predict(features)

        # Formater la réponse
        predicted_scores = {
            "Half_Time_Score": [int(predictions[0][0] / 2), int(predictions[0][1] / 2)],
            "Full_Time_Score": [int(predictions[0][0]), int(predictions[0][1])],
            "Goal_Times": match_data["Goal_Times"].values[0],
        }
        return predicted_scores
    except Exception as e:
        logging.error(f"Erreur lors de la prédiction des scores : {e}")
        return "Erreur lors de la prédiction des scores."

# === Commandes Telegram ===
def start(update, context):
    """Message de bienvenue."""
    update.message.reply_text("Bienvenue ! Utilisez la commande /predict <Équipe1> <Équipe2> pour obtenir les prédictions des scores.")

def predict(update, context):
    """Prédire les scores."""
    try:
        user_input = context.args
        if len(user_input) != 2:
            update.message.reply_text("Utilisez : /predict <Équipe1> <Équipe2>")
            return

        home_team, away_team = user_input
        predictions = predict_scores(home_team, away_team)
        if isinstance(predictions, str):
            update.message.reply_text(predictions)
        else:
            response = (
                f"Match : {home_team} vs {away_team}\n"
                f"Score à mi-temps : {predictions['Half_Time_Score'][0]}-{predictions['Half_Time_Score'][1]}\n"
                f"Score final : {predictions['Full_Time_Score'][0]}-{predictions['Full_Time_Score'][1]}\n"
                f"Moments des buts : {predictions['Goal_Times']}"
            )
            update.message.reply_text(response)
    except Exception as e:
        update.message.reply_text("Erreur lors du traitement de la commande.")
        logging.error(f"Erreur dans la commande /predict : {e}")

# === Configuration du bot ===
def main():
    updater = Updater(API_TOKEN, use_context=True)
    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("predict", predict))

    updater.start_polling()
    updater.idle()

if __name__ == "__main__":
    main()
<!---
Freddy85-iss/Freddy85-iss is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
