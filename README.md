<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Supra V12 Ultra Fix</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: sans-serif; touch-action: none; }
        
        /* Start-Zentrale */
        #launcher {
            position: absolute; top: 0; left: 0; width: 100vw; height: 100vh;
            display: flex; flex-direction: column; justify-content: center; align-items: center;
            background: #111; z-index: 9999;
        }

        #start-btn {
            width: 250px; height: 250px; background: #ce1126; border: 10px solid #fff;
            border-radius: 50%; color: white; font-size: 30px; font-weight: bold;
            cursor: pointer; box-shadow: 0 0 50px rgba(206,17,38,0.5);
        }

        /* Spiel-Elemente (Standardmäßig versteckt) */
        #game-ui { display: none; }
        
        #dash {
            position: absolute; top: 20px; left: 20px; color: #fff;
            background: rgba(0,0,0,0.8); padding: 15px; border-left: 5px solid #ce1126;
        }

        #controls {
            position: absolute; bottom: 30px; right: 40px;
            display: flex; align-items: flex-end; gap: 40px;
        }

        #pedal {
            width: 100px; height: 200px; background: #222; border-radius: 15px;
            border: 3px solid #444; display: flex; justify-content: center; align-items: flex-end;
            padding-bottom: 15px; touch-action: none;
        }

        #gas-pedal {
            width: 70px; height: 160px; background: linear-gradient(#888, #111);
            border-radius: 10px; transition: transform 0.05s; transform-origin: bottom;
        }
        #gas-pedal.active { transform: rotateX(-30deg) scaleY(0.9); }

        #shift {
            width: 110px; height: 110px; background: #ce1126; border-radius: 50%;
            border: 5px solid #fff; color: white; font-size: 22px; font-weight: bold;
        }
    </style>
</head>
<body>

    <div id="launcher">
        <button id="start-btn">START ENGINE<br><span style="font-size:12px;">(FULLSCREEN & SOUND)</span></button>
    </div>

    <div id="game-ui">
        <div id="dash">
            V12 RPM: <span id="n-rpm">0</span><br>
            GEAR: <span id="n-gear">1</span>
        </div>

        <div id="controls">
            <button id="shift">SHIFT</button>
            <div id="pedal"><div id="gas-pedal"></div></div>
        </div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.skypack.dev/three@0.136.0';

        let audioCtx, mainOsc, masterGain, active = false;
        let rpm = 0, gear = 1, isGas = false;

        const startBtn = document.getElementById('start-btn');

        startBtn.onclick = async () => {
            // 1. Fullscreen aktivieren
            try {
                const container = document.documentElement;
                if (container.requestFullscreen) await container.requestFullscreen();
                else if (container.webkitRequestFullscreen) await container.webkitRequestFullscreen();
                
                // Querformat versuchen
                if (screen.orientation && screen.orientation.lock) {
                    await screen.orientation.lock('landscape').catch(() => {});
                }
            } catch (err) { console.log("FS Blocked"); }

            // 2. Audio Engine starten
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            
            // "Unlock" Sound (Stille)
            const buffer = audioCtx.createBuffer(1, 1, 22050);
            const source = audioCtx.createBufferSource();
            source.buffer = buffer;
            source.connect(audioCtx.destination);
            source.start(0);

            // V12 Bass Setup
            masterGain = audioCtx.createGain();
            masterGain.gain.value = 5.0; // MAX VOLUME

            const filter = audioCtx.createBiquadFilter();
            filter.type = 'lowpass';
            filter.frequency.value = 500;

            mainOsc = audioCtx.createOscillator();
            mainOsc.type = 'sawtooth';
            mainOsc.connect(filter);
            filter.connect(masterGain);
            masterGain.connect(audioCtx.destination);
            mainOsc.start();

            // 3. UI umschalten
            document.getElementById('launcher').style.display = 'none';
            document.getElementById('game-ui').style.display = 'block';
            active = true;
            rpm = 850;
            animate();
        };

        // Einfaches 3D (damit man sieht dass es läuft)
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x050505);
        const camera = new THREE.PerspectiveCamera(45, window.innerWidth/window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer();
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);
        camera.position.set(10, 5, -10); camera.lookAt(0,0,0);

        // Steuerung
        const pedal = document.getElementById('pedal');
        const gasInner = document.getElementById('gas-pedal');

        const gasAn = (e) => { e.preventDefault(); isGas = true; gasInner.classList.add('active'); };
        const gasAus = (e) => { e.preventDefault(); isGas = false; gasInner.classList.remove('active'); };

        pedal.addEventListener('touchstart', gasAn);
        pedal.addEventListener('mousedown', gasAn);
        window.addEventListener('touchend', gasAus);
        window.addEventListener('mouseup', gasAus);

        document.getElementById('shift').onclick = () => {
            gear = gear < 6 ? gear + 1 : 1;
            document.getElementById('n-gear').innerText = gear;
            rpm -= 4000;
        };

        function animate() {
            requestAnimationFrame(animate);
            if(active) {
                let target = isGas ? 9500 : 850;
                rpm += (target - rpm) * 0.1;
                document.getElementById('n-rpm').innerText = Math.floor(rpm);
                
                // Sound-Update (Extrem Tief)
                mainOsc.frequency.setTargetAtTime(4 + (rpm/250), audioCtx.currentTime, 0.05);
            }
            renderer.render(scene, camera);
        }
    </script>
</body>
</html>
