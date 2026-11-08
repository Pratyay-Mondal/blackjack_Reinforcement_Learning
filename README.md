

# Blackjack RL: Entscheidungsfindung unter stochastischer Unsicherheit 

- Entwicklung einer modularen Simulationsumgebung in reinem Python (ohne externe RL-Frameworks) zur Modellierung stochastischer Markov-Entscheidungsprozesse (MDP) nach strikten Nevada-Casino-Regeln (Hit, Stand, Double Down, Split, Late Surrender, Dealer H17). 
- Implementierung und vergleichende Analyse tabellarischer Temporal-Difference-Verfahren: Off-Policy Q- Learning (Bellman-Optimalit¨at) vs. On-Policy SARSA (ϵ-Greedy-Policy-Verbesserung) ¨uber mehr als 106 Trainings- und Evaluationsepisoden. 
- Integration eines Hi-Lo-True-Count-Kartenz¨ahlsystems mit dynamischer Wetteinsatz-Skalierung (bis zu 10x): Untersuchung des Fluchs der Dimensionalit¨at bei Zustandsexpansion auf 9.000 Zustands- Aktions-Paare sowie empirischer Nachweis der SARSA-Resilienz gegen Q-Learnings Maximization Bias. 
- Technologien: Python (From-Scratch), Reinforcement Learning (Q-Learning, SARSA), Markov-Entscheidungsprozesse (MDP), Stochastische Modellierung, NumPy, Matplotlib.

