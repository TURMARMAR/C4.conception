<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Split the G - Télécommande</title>
    <style>
        :root { --fond: #0b1319; --carte: #14222c; --cyan: #00e5ff; --vert: #2ecc71; --rouge: #ff4757; }
        * { box-sizing: border-box; }
        body {
            font-family: monospace; background: var(--fond); color: var(--cyan);
            margin: 0; padding: 12px; min-height: 100vh;
            display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 10px;
            touch-action: manipulation; overscroll-behavior-y: none; user-select: none; -webkit-user-select: none;
        }
        h2 { margin: 0; letter-spacing: 2px; text-transform: uppercase; font-size: 1.2rem; }
        .bloc { width: 100%; max-width: 420px; }
        .carte { background: var(--carte); border: 2px solid var(--cyan); border-radius: 12px; padding: 10px 14px; }
        .entete, .reperes, .ligne-trim, .haut { display: flex; justify-content: space-between; align-items: center; }
        .entete { letter-spacing: 2px; }
        .entete strong { font-size: 1.2rem; }
        .reperes { font-size: .75rem; opacity: .85; }
        .ligne-trim { margin-top: 8px; gap: 10px; }
        .ligne-trim strong { min-width: 3.5em; text-align: center; font-size: 1.1rem; }
        .ligne-trim span { flex: 1; letter-spacing: 2px; font-size: .85rem; }
        .haut { gap: 10px; align-items: stretch; }
        .haut button { flex: 1; }
        #mesure { display: flex; flex-direction: column; justify-content: center; text-align: right; min-width: 7.5em; padding: 6px 12px; }
        #mesure span { font-size: .65rem; letter-spacing: 1px; }
        #mesure strong { font-size: 1.2rem; }
        #message { min-height: 1.1em; margin: 0; font-size: .8rem; color: var(--rouge); text-align: center; }

        button {
            background: var(--carte); color: var(--cyan); border: 2px solid var(--cyan); border-radius: 12px;
            padding: 12px 10px; font: bold 1rem monospace; cursor: pointer;
        }
        button:active:not(:disabled) { background: var(--cyan); color: var(--fond); }
        button:disabled { opacity: .4; }
        #btn-connect.ok { color: var(--vert); border-color: var(--vert); background: rgba(46, 204, 113, .1); opacity: 1; }
        #btn-centre { padding: 6px 12px; font-size: .75rem; }
        .ligne-trim button { width: 56px; padding: 6px 0; font-size: 1.2rem; }
        #btn-stop { padding: 20px 10px; color: var(--rouge); border-color: var(--rouge); background: rgba(255, 71, 87, .1); }
        #btn-stop:active { background: var(--rouge); color: #fff; }

        /* Curseurs : grosse poignée pour le pouce */
        input[type=range] { -webkit-appearance: none; appearance: none; width: 100%; height: 46px; margin: 0; background: transparent; }
        input[type=range]:disabled { opacity: .4; }
        input[type=range]::-webkit-slider-runnable-track {
            height: 14px; border-radius: 7px; background: var(--fond); box-shadow: inset 0 0 0 2px var(--cyan);
        }
        input[type=range]::-webkit-slider-thumb {
            -webkit-appearance: none; width: 44px; height: 44px; margin-top: -15px;
            border-radius: 50%; background: var(--cyan); border: 4px solid var(--fond);
        }
    </style>
</head>
<body>

    <h2>Split the G</h2>

    <div class="bloc haut">
        <button id="btn-connect">🔌 CONNECTER</button>
        <div id="mesure" class="carte"><span>TENSION MOTEUR</span><strong id="aff-tension">-- V</strong></div>
    </div>

    <div class="bloc carte">
        <div class="entete"><span>DIRECTION</span><strong id="aff-direction">Centre</strong></div>
        <input type="range" id="curseur-direction" min="45" max="135" step="5" value="90" aria-label="Direction" disabled>
        <div class="reperes"><span>◄ BÂBORD</span><button id="btn-centre" disabled>CENTRER</button><span>TRIBORD ►</span></div>
        <div class="ligne-trim">
            <span>TRIM</span>
            <button id="btn-trim-moins" aria-label="Trim vers bâbord" disabled>−</button>
            <strong id="aff-trim">0°</strong>
            <button id="btn-trim-plus" aria-label="Trim vers tribord" disabled>+</button>
        </div>
    </div>

    <div class="bloc carte">
        <div class="entete"><span>VITESSE</span><strong id="aff-vitesse">Arrêt</strong></div>
        <input type="range" id="curseur-vitesse" min="-100" max="100" step="5" value="0" aria-label="Vitesse" disabled>
        <div class="reperes"><span>◄ ARRIÈRE</span><span>0</span><span>AVANT ►</span></div>
    </div>

    <button id="btn-stop" class="bloc">ARRÊT D'URGENCE</button>
    <p id="message" class="bloc" role="status"></p>

    <script>
        const UART_SERVICE_UUID = "6e400001-b5a3-f393-e0a9-e50e24dcca9e";
        const ANGLE_CENTRE = 90;     // mêmes valeurs que dans le code micro:bit
        const TRIM_MAX = 15;
        const PAUSE_MS = 40;         // délai mini entre deux envois Bluetooth
        const SIGNAL_VIE_MS = 400;   // la micro:bit coupe les moteurs après 1,5 s de silence

        const $ = id => document.getElementById(id);
        const btnConnect = $('btn-connect'), btnCentre = $('btn-centre'), btnStop = $('btn-stop');
        const btnTrimMoins = $('btn-trim-moins'), btnTrimPlus = $('btn-trim-plus');
        const curseurDirection = $('curseur-direction'), curseurVitesse = $('curseur-vitesse');
        const pause = ms => new Promise(r => setTimeout(r, ms));

        let caracEcriture = null;    // téléphone -> micro:bit
        let trim = lireTrimMemorise();

        // --- ENVOI BLUETOOTH ---
        // Un seul message à la fois, et seule la DERNIÈRE valeur de chaque canal est envoyée :
        // on peut glisser le doigt vite sans saturer la liaison.
        // Protocole : "D<angle>" direction, "V<-100 à 100>" vitesse, "T<degrés>" trim, "S" arrêt, "P" signal de vie.
        const PRIORITE = ['S', 'V', 'T', 'D', 'P'];   // l'arrêt passe toujours en premier
        const aEnvoyer = new Map();                   // canal -> dernier message en attente
        let envoiEnCours = false;

        function envoyer(canal, message) {
            aEnvoyer.set(canal, message);
            viderFile();
        }

        async function viderFile() {
            if (envoiEnCours || !caracEcriture) return;
            envoiEnCours = true;
            while (aEnvoyer.size > 0 && caracEcriture) {
                const canal = PRIORITE.find(c => aEnvoyer.has(c));
                const message = aEnvoyer.get(canal);
                aEnvoyer.delete(canal);
                try {
                    await caracEcriture.writeValue(new TextEncoder().encode(message + "\n"));
                } catch (e) {
                    console.log("Erreur d'envoi :", e);
                    // On réessaie, sauf si un ordre plus récent ou un arrêt remplace celui-ci
                    if (caracEcriture && !aEnvoyer.has(canal) && !aEnvoyer.has('S')) aEnvoyer.set(canal, message);
                }
                await pause(PAUSE_MS);
            }
            envoiEnCours = false;
        }

        // --- RÉCEPTION BLUETOOTH ---
        // Protocole : "U<volts>" tension moteur, "X" moteurs coupés après une perte de liaison.
        let tampon = "";

        function surReception(evenement) {
            tampon += new TextDecoder().decode(evenement.target.value);
            const lignes = tampon.split("\n");
            tampon = lignes.pop();                    // la fin peut être incomplète
            lignes.forEach(ligne => traiterLigne(ligne.trim()));
        }

        function traiterLigne(ligne) {
            if (ligne[0] === 'U') {
                const volts = parseFloat(ligne.slice(1));
                if (!isNaN(volts)) $('aff-tension').textContent = volts.toFixed(1) + " V";
            } else if (ligne === 'X') {
                arretMoteurs();
                afficherMessage("Liaison interrompue : moteurs coupés");
            }
        }

        // --- AFFICHAGE ---
        function afficherMessage(texte) {
            $('message').textContent = texte;
        }

        function majAffichage() {
            const ecart = Number(curseurDirection.value) - ANGLE_CENTRE;
            const vitesse = Number(curseurVitesse.value);
            $('aff-direction').textContent = ecart === 0 ? "Centre" : (ecart < 0 ? "Bâbord " : "Tribord ") + Math.abs(ecart) + "°";
            $('aff-vitesse').textContent = vitesse === 0 ? "Arrêt" : (vitesse < 0 ? "Arrière " : "Avant ") + Math.abs(vitesse) + " %";
            $('aff-trim').textContent = (trim > 0 ? "+" : "") + trim + "°";
        }

        function etatConnexion(connecte) {
            btnConnect.disabled = connecte;
            btnConnect.classList.toggle('ok', connecte);
            btnConnect.textContent = connecte ? "✅ CONNECTÉ" : "🔌 CONNECTER";
            [curseurDirection, curseurVitesse, btnCentre, btnTrimMoins, btnTrimPlus].forEach(e => e.disabled = !connecte);
        }

        function remiseAZero() {
            curseurDirection.value = ANGLE_CENTRE;
            curseurVitesse.value = 0;
            majAffichage();
        }

        // --- TRIM (conservé sur le téléphone entre deux connexions) ---
        function lireTrimMemorise() {
            try {
                const valeur = parseInt(localStorage.getItem('trim'), 10);
                return isNaN(valeur) ? 0 : Math.max(-TRIM_MAX, Math.min(TRIM_MAX, valeur));
            } catch (e) {
                return 0;
            }
        }

        function changerTrim(ecart) {
            trim = Math.max(-TRIM_MAX, Math.min(TRIM_MAX, trim + ecart));
            try { localStorage.setItem('trim', trim); } catch (e) { /* sans mémoire : le trim reste valable ici */ }
            majAffichage();
            envoyer('T', 'T' + trim);
        }

        // --- COMMANDES ---
        function arretMoteurs() {
            curseurVitesse.value = 0;
            majAffichage();
            aEnvoyer.delete('V');   // inutile d'envoyer une ancienne vitesse après l'arrêt
            envoyer('S', 'S');
        }

        curseurDirection.addEventListener('input', () => {
            majAffichage();
            envoyer('D', 'D' + curseurDirection.value);
        });

        curseurVitesse.addEventListener('input', () => {
            afficherMessage("");
            majAffichage();
            envoyer('V', 'V' + curseurVitesse.value);
        });

        btnCentre.addEventListener('click', () => {
            curseurDirection.value = ANGLE_CENTRE;
            majAffichage();
            envoyer('D', 'D' + ANGLE_CENTRE);
        });

        btnTrimMoins.addEventListener('click', () => changerTrim(-1));
        btnTrimPlus.addEventListener('click', () => changerTrim(1));

        // pointerdown : réagit dès le contact du doigt, sans attendre le relâchement
        btnStop.addEventListener('pointerdown', arretMoteurs);

        // Page masquée (changement d'application, écran verrouillé) : on coupe les moteurs
        document.addEventListener('visibilitychange', () => {
            if (document.hidden && caracEcriture) arretMoteurs();
        });

        // Signal de vie : prouve à la micro:bit que la page fonctionne toujours
        setInterval(() => {
            if (caracEcriture && !aEnvoyer.has('P')) envoyer('P', 'P');
        }, SIGNAL_VIE_MS);

        // --- CONNEXION ---
        function surDeconnexion() {
            caracEcriture = null;
            envoiEnCours = false;
            aEnvoyer.clear();
            tampon = "";
            $('aff-tension').textContent = "-- V";
            remiseAZero();
            etatConnexion(false);
            afficherMessage("Déconnecté : moteurs coupés");
        }

        btnConnect.addEventListener('click', async () => {
            if (!navigator.bluetooth) {
                afficherMessage("Bluetooth indisponible : ouvre la page dans Bluefy ou Chrome");
                return;
            }
            let appareil = null;
            btnConnect.disabled = true;
            afficherMessage("");
            try {
                appareil = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC micro:bit' }],
                    optionalServices: [UART_SERVICE_UUID]
                });
                appareil.addEventListener('gattserverdisconnected', surDeconnexion, { once: true });
                const serveur = await appareil.gatt.connect();
                const service = await serveur.getPrimaryService(UART_SERVICE_UUID);

                // On choisit chaque canal d'après ce qu'il sait faire, pas d'après son numéro :
                // cela évite l'erreur "GATT operation not permitted".
                const caracteristiques = await service.getCharacteristics();
                const ecriture = caracteristiques.find(c => c.properties.write || c.properties.writeWithoutResponse);
                const lecture = caracteristiques.find(c => c.properties.notify || c.properties.indicate);
                if (!ecriture) throw new Error("canal d'écriture introuvable");

                // Sans réception, le pilotage reste possible : seule la mesure manque
                try {
                    if (lecture) {
                        lecture.addEventListener('characteristicvaluechanged', surReception);
                        await lecture.startNotifications();
                    }
                } catch (e) {
                    console.log("Réception indisponible :", e);
                    afficherMessage("Connecté, mais mesure indisponible");
                }

                caracEcriture = ecriture;
                remiseAZero();
                etatConnexion(true);
                arretMoteurs();                       // micro:bit et curseurs repartent d'accord
                envoyer('T', 'T' + trim);
                envoyer('D', 'D' + ANGLE_CENTRE);
            } catch (erreur) {
                btnConnect.disabled = false;
                if (appareil) {
                    if (appareil.gatt.connected) appareil.gatt.disconnect();
                    afficherMessage("Connexion impossible : " + erreur.message);
                } else if (erreur.name !== 'NotFoundError') {     // NotFoundError ici = choix annulé
                    afficherMessage("Connexion impossible : " + erreur.message);
                }
            }
        });

        majAffichage();
    </script>
</body>
</html>
