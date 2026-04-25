# Multi-agent-disaster-response-system
The Multi-Agent Disaster Response System is an advanced AI-driven platform designed to simulate and optimize emergency response during natural or man-made disasters. The system leverages multiple intelligent agents that collaborate in real time to analyze risks, allocate resources, and plan rescue operations efficiently.
from fastapi import FastAPI, WebSocket
from fastapi.responses import HTMLResponse
from pydantic import BaseModel
import json
import asyncio
import time

app = FastAPI()

# ------------------ ENHANCED AGENTS ------------------
class AgentResponse:
    def __init__(self):
        self.responses = []

    def add(self, agent, data):
        self.responses.append({"agent": agent, "data": data, "timestamp": time.time()})

def risk_agent(disaster):
    disaster = disaster.lower()
    risks = {
        "flood": {"risk": "HIGH", "areas": ["Zone A", "Zone B"], "severity": 7},
        "earthquake": {"risk": "CRITICAL", "areas": ["Zone C", "Zone D"], "severity": 10},
        "fire": {"risk": "HIGH", "areas": ["Zone E", "Zone F"], "severity": 8},
        "hurricane": {"risk": "CRITICAL", "areas": ["Zone G", "Zone H"], "severity": 9}
    }
    return risks.get(disaster, {"risk": "MEDIUM", "areas": ["Zone X"], "severity": 4})

def resource_agent(risk):
    resources = {
        "CRITICAL": {"ambulances": 12, "teams": 6, "helicopters": 3},
        "HIGH": {"ambulances": 8, "teams": 4, "helicopters": 1},
        "MEDIUM": {"ambulances": 4, "teams": 2, "helicopters": 0}
    }
    return resources.get(risk, {"ambulances": 2, "teams": 1, "helicopters": 0})

def logistics_agent(areas):
    return [{"area": area, "eta": f"{len(area)*10}min", "priority": "HIGH"} for area in areas]

def medical_agent(resources):
    patients = resources["ambulances"] * 5
    return {"priority_cases": patients // 2, "total_capacity": patients}

def evac_agent(areas):
    return sum(len(area) * 1000 for area in areas)

# ------------------ API ------------------
class Input(BaseModel):
    disaster: str

@app.get("/", response_class=HTMLResponse)
def home():
    return """
<!DOCTYPE html>
<html>
<head>
    <title>🚨 AI Disaster Command Center</title>
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        * { margin:0; padding:0; box-sizing:border-box; }
        body { 
            background: linear-gradient(45deg, #000, #001122, #000); 
            color: #00ff88; 
            font-family: 'Courier New', monospace;
            overflow-x: hidden;
        }
        .header { 
            background: rgba(0,255,136,0.1); 
            padding: 20px; 
            text-align: center; 
            border-bottom: 2px solid #00ff88;
            box-shadow: 0 0 30px rgba(0,255,136,0.3);
        }
        .controls { padding: 20px; text-align: center; }
        select, button { 
            padding: 15px 25px; 
            margin: 10px; 
            background: linear-gradient(45deg, #00ff88, #00cc66);
            border: none; 
            border-radius: 25px; 
            font-size: 16px; 
            cursor: pointer;
            box-shadow: 0 5px 15px rgba(0,255,136,0.4);
            transition: all 0.3s;
        }
        button:hover { transform: scale(1.05); box-shadow: 0 8px 25px rgba(0,255,136,0.6); }
        .dashboard { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; padding: 20px; max-width: 1400px; margin: auto; }
        .card { 
            background: rgba(0,20,40,0.9); 
            border: 2px solid #00ff88; 
            border-radius: 15px; 
            padding: 20px; 
            backdrop-filter: blur(10px);
            box-shadow: 0 10px 40px rgba(0,255,136,0.2);
        }
        .map-card { height: 500px; }
        #map { height: 100%; border-radius: 10px; }
        pre { background: rgba(0,0,0,0.5); padding: 15px; border-radius: 10px; overflow: auto; max-height: 300px; }
        .progress { background: rgba(0,255,136,0.2); border-radius: 10px; height: 20px; margin: 10px 0; overflow: hidden; }
        .progress-bar { height: 100%; background: linear-gradient(90deg, #00ff88, #00cc66); transition: width 2s; }
        .particles { position: fixed; top: 0; left: 0; pointer-events: none; z-index: 1000; }
        @media (max-width: 768px) { .dashboard { grid-template-columns: 1fr; } }
    </style>
</head>
<body>
    <div class="particles" id="particles"></div>
    
    <div class="header">
        <h1>🚨 AI Multi-Agent Disaster Command Center</h1>
        <p>Real-time Response Coordination | 6 Intelligent Agents</p>
    </div>

    <div class="controls">
        <select id="disaster">
            <option value="flood">🌊 Flood</option>
            <option value="earthquake">🌍 Earthquake</option>
            <option value="fire">🔥 Fire</option>
            <option value="hurricane">🌪️ Hurricane</option>
        </select>
        <button onclick="runSimulation()">🚀 DEPLOY AGENTS</button>
        <button onclick="toggleTheme()">🌙 Dark/Light</button>
    </div>

    <div class="dashboard">
        <div class="card">
            <h3>📡 Agent Responses</h3>
            <pre id="output">Select disaster and click DEPLOY AGENTS</pre>
        </div>
        
        <div class="card">
            <h3>📊 Resource Dashboard</h3>
            <canvas id="chart" width="400" height="200"></canvas>
            <div id="progress-container"></div>
        </div>

        <div class="card map-card">
            <h3>🗺️ Live Deployment Map</h3>
            <div id="map"></div>
        </div>

        <div class="card">
            <h3>⚡ Live Metrics</h3>
            <div id="metrics">
                <div>👥 Evacuated: <span id="evac">0</span></div>
                <div>🚑 Ambulances: <span id="ambulances">0</span></div>
                <div>👨‍🚒 Teams: <span id="teams">0</span></div>
                <div>🚁 Helicopters: <span id="helicopters">0</span></div>
            </div>
        </div>
    </div>

    <script>
        let map, chart;
        const particles = document.getElementById('particles');
        
        // Initialize map
        function initMap() {
            map = L.map('map').setView([40.7128, -74.0060], 10);
            L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
        }

        // Particle explosion effect
        function createParticles(x, y) {
            for(let i = 0; i < 20; i++) {
                const particle = document.createElement('div');
                particle.style.cssText = `
                    position: absolute; left: ${x}px; top: ${y}px;
                    width: 4px; height: 4px; background: #00ff88;
                    border-radius: 50%; pointer-events: none;
                    animation: explode 1s ease-out forwards;
                `;
                particles.appendChild(particle);
                setTimeout(() => particle.remove(), 1000);
            }
        }

        // Chart setup
        function initChart() {
            const ctx = document.getElementById('chart').getContext('2d');
            chart = new Chart(ctx, {
                type: 'doughnut',
                data: { labels: ['Ambulances', 'Teams', 'Helicopters'], datasets: [{ data: [0,0,0], backgroundColor: ['#00ff88', '#00cc66', '#0088ff'] }] },
                options: { responsive: true, plugins: { legend: { labels: { color: '#00ff88' } } } }
            });
        }

        // Main simulation
        async function runSimulation() {
            const disaster = document.getElementById('disaster').value;
            
            // Clear previous markers
            map.eachLayer(layer => { if (layer instanceof L.Marker) map.removeLayer(layer); });
            
            // Simulate agent chain with delays
            const response = new AgentResponse();
            
            // Risk Agent
            await new Promise(r => setTimeout(r, 500));
            const risk = risk_agent(disaster);
            response.add('Risk Agent', risk);
            addMapMarkers(risk.areas, 'red');
            
            // Resource Agent
            await new Promise(r => setTimeout(r, 800));
            const resources = resource_agent(risk.risk);
            response.add('Resource Agent', resources);
            
            // Logistics Agent
            await new Promise(r => setTimeout(r, 1200));
            const logistics = logistics_agent(risk.areas);
            response.add('Logistics Agent', logistics);
            
            // Medical Agent
            await new Promise(r => setTimeout(r, 1500));
            const medical = medical_agent(resources);
            response.add('Medical Agent', medical);
            
            // Evac Agent
            const evac = evac_agent(risk.areas);
            response.add('Evac Agent', {total: evac});
            
            displayResults(response.responses, resources, evac);
            createParticles(window.innerWidth/2, 200);
            
            // Sound effect (silent fallback)
            new Audio('data:audio/wav;base64,UklGRnoGAABXQVZFZm10IBAAAAABAAEAQB8AAEAfAAABAAgAZGF0YQoGAACBhYqFbF1fdJivrJBhNjVgodDbq2EcBj+a2/LDciUFLIHO8tiJNwgZaLvt559NEAxQp+PwtmMcBwB').play().catch(() => {});
        }

        function risk_agent(disaster) {
            const risks = {
                "flood": {"risk": "HIGH", "areas": ["Zone A", "Zone B"], "severity": 7},
                "earthquake": {"risk": "CRITICAL", "areas": ["Zone C", "Zone D"], "severity": 10},
                "fire": {"risk": "HIGH", "areas": ["Zone E", "Zone F"], "severity": 8},
                "hurricane": {"risk": "CRITICAL", "areas": ["Zone G", "Zone H"], "severity": 9}
            };
            return risks[disaster] || {"risk": "MEDIUM", "areas": ["Zone X"], "severity": 4};
        }

        function resource_agent(risk) {
            const resources = {
                "CRITICAL": {"ambulances": 12, "teams": 6, "helicopters": 3},
                "HIGH": {"ambulances": 8, "teams": 4, "helicopters": 1},
                "MEDIUM": {"ambulances": 4, "teams": 2, "helicopters": 0}
            };
            return resources[risk] || {"ambulances": 2, "teams": 1, "helicopters": 0};
        }

        function addMapMarkers(areas, color) {
            const coords = [[40.7,-74.0], [40.8,-73.9], [40.75,-74.1], [40.65,-73.95], [40.85,-73.85], [40.9,-74.05], [40.6,-73.9], [40.55,-74.15]];
            areas.slice(0, coords.length).forEach((area, i) => {
                L.marker(coords[i]).addTo(map)
                    .bindPopup(`<b>${area}</b><br>Priority Response`)
                    .openPopup();
                createParticles(100 + i*100, 300 + i*20);
            });
        }

        function displayResults(responses, resources, evac) {
            document.getElementById('output').innerText = 
                responses.map(r => `${r.agent}:\n${JSON.stringify(r.data, null, 2)}`).join('\\n\\n');
            
            chart.data.datasets[0].data = [resources.ambulances, resources.teams, resources.helicopters];
            chart.update();
            
            // Update metrics
            document.getElementById('evac').textContent = evac.toLocaleString();
            document.getElementById('ambulances').textContent = resources.ambulances;
            document.getElementById('teams').textContent = resources.teams;
            document.getElementById('helicopters').textContent = resources.helicopters;
            
            // Animated progress bars
            const container = document.getElementById('progress-container');
            container.innerHTML = `
                <div>Deployment Progress</div>
                <div class="progress"><div class="progress-bar" style="width:0%" id="prog1"></div></div>
                <div class="progress"><div class="progress-bar" style="width:0%" id="prog2"></div></div>
            `;
            setTimeout(() => {
                document.getElementById('prog1').style.width = '85%';
                document.getElementById('prog2').style.width = '92%';
            }, 100);
        }

        function toggleTheme() {
            document.body.classList.toggle('light');
        }

        // Initialize
        initMap();
        initChart();
        
        // CSS for particles
        const style = document.createElement('style');
        style.textContent = `
            @keyframes explode {
                0% { transform: scale(1) translate(0,0); opacity: 1; }
                100% { transform: scale(0) translate(var(--dx), var(--dy)); opacity: 0; }
            }
            .light { background: linear-gradient(45deg, #f0f8ff, #e6f3ff); color: #0066cc !important; }
            .light .card { background: rgba(255,255,255,0.95); border-color: #0066cc; }
        `;
        document.head.appendChild(style);
    </script>
</body>
</html>
"""

@app.post("/simulate")
def simulate(input: Input):
    risk = risk_agent(input.disaster)
    resources = resource_agent(risk["risk"])
    logistics = logistics_agent(risk["areas"])
    medical = medical_agent(resources)
    evac = evac_agent(risk["areas"])
    
    return {
        "risk": risk,
        "resources": resources,
        "logistics": logistics,
        "medical": medical,
        "evacuation": evac,
        "status": "DEPLOYED"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
