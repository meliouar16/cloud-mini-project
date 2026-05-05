# Réponses aux questions :

### Traces > Compréhension

Commenter, expliquer les attributs (http.method, http.route, db.statement, etc ...)

Réponse :
La trace suit bien la chaîne api-gateway → task-service → postgres. Les attributs permettent de comprendre le déroulement de la requête
On peux voir que ces attributs nous donnes des détails sur le contenu des requêtes par exemple dans db.statement on peux voir la requête insert qui a été éxecuté, on à des infos sur la méthode utilisé, suur quel route ça été lancer (http.url & target) etc..

#### Visualisation

1. Quelle syntaxe LogQL est utilisée ?
2. Quelle différence y a-t-il avec une requête Prometheus ?
   Déclencher une erreur volontairement (ex: créer une tâche sans title). Retrouver le log d'erreur correspondant dans Loki.
3. Quelle requête utiliser pour filtrer ?

4. Écrire une requête LogQL qui affiche uniquement les logs de niveau error sur tous les services à la fois.

5. (b) Écrire une requête qui extrait et filtre sur le champ statusCode pour ne voir que les requêtes ayant retourné un 500.

6. Dans Loki, comment obtenir l'équivalent en passant par les logs ?

7. Entre ces deux approches, laquelle est la plus adaptée et pourquoi ?

8. Peut-on retrouver ce traceId dans les logs Loki ?

9. Que faudrait-il configurer pour que ce soit automatique ?

### Réponses :

1. LogQL est écrit en objet avec des pipelines

2. LogQL interroge des logs (texte/JSON, filtres de contenu), alors que PromQL interroge des métriques numériques déjà agrégées (counters, gauges, histograms).

3. {service_name="/cloud-mini-project-task-service-1", level="error"} |= ``

4. {level="error"} | json

5. (b) {level="error"} | json | res_statusCode >= 500

6.

sum(
count_over_time(
{level="error"} | json | res_statusCode >= 500
[5m]
)
)

6. Approche la plus adaptée :

- Pour mesurer le volume/taux d'erreurs dans le temps, Prometheus est plus adapté (plus fiable pour les métriques et l'alerting).
- Pour comprendre le détail d'une erreur précise, Loki est plus adapté (message, contexte, payload, stack trace).

8. TraceId après un POST /api/tasks :

- Oui, on retrouve la traceId dans loki.
- Ce traceId n'est pas présent automatiquement dans Loki sans corrélation explicite dans les logs applicatifs.

9. Configuration nécessaire pour corrélation automatique logs <-> traces :

- Injecter `trace_id` et `span_id` dans les logs applicatifs (pino + contexte OpenTelemetry).
- Parser ces champs dans Promtail (`| json`) et les conserver dans les logs Loki.
- Activer la corrélation dans Grafana pour naviguer de Loki vers Tempo.

10. Démarche d'investigation en cas de pic d'erreurs Prometheus :

- Métriques (Prometheus) : identifier le service, la période et le type d'erreur (ex: 5xx).

- Logs (Loki) : filtrer les logs `level="error"` du service concerné et isoler les `statusCode >= 500`.

- Traces (Tempo) : ouvrir les traces en erreur sur la même fenêtre temporelle et analyser la waterfall pour localiser l'étape fautive.
