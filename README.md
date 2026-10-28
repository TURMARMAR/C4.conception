<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Split the G - Interface de Contrôle</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="dashboard">
        <header>
            <h1>SPLIT THE G <span class="team-badge">TEAM C4</span></h1>
            <div class="status">Statut: <span class="status-indicator online">En attente de liaison</span></div>
        </header>

        <div class="telemetry">
            <div class="panel">
                <h2>VITESSE (M1)</h2>
                <div class="value-display" id="speed-display">0</div>
                <div class="gauge-container">
                    <div class="gauge-bar speed-gauge" id="speed-bar"></div>
                </div>
            </div>
            
            <div class="panel">
                <h2>ANGLE BARRE</h2>
                <div class="value-display" id="angle-display">0°</div>
                <div class="rudder-container">
                    <div class="boat-hull"></div>
                    <div class="rudder" id="rudder-line"></div>
                </div>
            </div>
        </div>

        <div class="controls">
            <div class="control-group">
                <h3>MOTEUR</h3>
                <button class="btn btn-action" id="btn-accel">ACCÉLÉRER (+)</button>
                <button class="btn btn-danger" id="btn-stop">ARRÊT URGENCE</button>
                <button class="btn btn-action" id="btn-decel">DÉCÉLÉRER (-)</button>
            </div>

            <div class="control-group">
                <h3>DIRECTION (Pas de 5°)</h3>
                <div class="steer-buttons">
                    <button class="btn btn-steer" id="btn-port">◀ BÂBORD</button>
                    <button class="btn btn-steer" id="btn-stbd">TRIBORD ▶</button>
                </div>
                <h3>TRIM</h3>
                <div class="steer-buttons">
                    <button class="btn btn-trim" id="btn-trim-port">-1°</button>
                    <span id="trim-display" class="trim-value">0°</span>
                    <button class="btn btn-trim" id="btn-trim-stbd">+1°</button>
                </div>
            </div>
        </div>
    </div>

    <!-- Le script est chargé à la fin pour que le HTML s'affiche d'abord -->
    <script src="script.js"></script>
</body>
</html>
