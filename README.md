<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FS22 Live Server Data</title>
    <!-- Load Tailwind CSS via CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Load jQuery (required for the widget's JS logic) -->
    <script src="https://code.jquery.com/jquery-3.7.1.min.js"></script>
    <!-- Set default font -->
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #1f2937;
        }
        /* Custom styles for the widget appearance */
        #fs22-status-widget {
            background: #2b2b2b;
            color: #f0f0f0;
            padding: 1.5rem;
            border-radius: 0.75rem;
            line-height: 1.6;
            margin-bottom: 1rem;
        }
        #fs22-status-widget h4 {
            color: #a8ffb5;
            margin-top: 0;
            margin-bottom: 0.3rem;
            font-weight: 700;
            font-size: 1.1em;
        }
        #fs22-status-widget .status-dot {
            height: 10px;
            width: 10px;
            background-color: #e74c3c; /* Red (Offline) */
            border-radius: 50%;
            display: inline-block;
            margin-right: 5px;
        }
        #fs22-status-widget.online .status-dot {
            background-color: #2ecc71; /* Green (Online) */
        }
        .loading-status, .error-status {
            color: #f1c40f;
            font-style: italic;
        }
        .critical-error {
            color: #ff5555;
            font-weight: 700;
        }

        /* Map specific styles */
        #map-container {
            position: relative;
            width: 100%;
            padding-top: 75%; /* 4:3 Aspect Ratio for map */
            overflow: hidden;
            border-radius: 0.5rem;
        }
        #map-image {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            object-fit: cover;
            border-radius: 0.5rem;
        }
        .data-snippet-box {
            max-height: 120px;
            overflow-y: auto;
            white-space: pre-wrap;
            font-family: monospace;
            font-size: 0.65rem;
            background: #111827;
            padding: 8px;
            border-radius: 4px;
            color: #a8ffb5;
        }
    </style>
</head>
<body class="p-4 sm:p-8 flex justify-center items-start min-h-screen">

    <div class="w-full max-w-4xl space-y-4">
        <h1 class="text-3xl font-bold text-gray-100 mb-6">FS22 Live Server Data (Crossplay Compatible)</h1>

        <!-- Live Status Widget (Top Bar) -->
        <div id="fs22-status-widget" class="shadow-xl">
            <p class="loading-status">Loading server status...</p>
        </div>

        <!-- Dashboard Content Grid -->
        <div class="grid grid-cols-1 lg:grid-cols-3 gap-4">
            
            <!-- Map Container (Left Panel) -->
            <div class="lg:col-span-2 p-4 bg-gray-800 rounded-xl shadow-xl">
                <h3 class="text-xl font-bold text-gray-100 mb-2">Live Map Image</h3>
                <div id="map-container">
                    <img id="map-image" src="" alt="Loading Map..." class="animate-pulse">
                    <p id="map-loading" class="absolute inset-0 flex items-center justify-center text-white bg-gray-900/50">Loading map data...</p>
                </div>
            </div>

            <!-- Saved Data Display (Right Panel) -->
            <div class="lg:col-span-1 p-4 bg-gray-800 rounded-xl shadow-xl space-y-4">
                <h3 class="text-xl font-bold text-gray-100 mb-4">Saved Game Data Snippets</h3>
                
                <!-- Career Data -->
                <div>
                    <h4 class="text-lg font-semibold text-gray-300">Career Save Data:</h4>
                    <p id="career-status" class="text-yellow-400 text-xs mt-1">Fetching saved career data...</p>
                    <div id="career-data-display" class="data-snippet-box"></div>
                </div>

                <!-- Vehicles Data -->
                <div>
                    <h4 class="text-lg font-semibold text-gray-300">Vehicles Data:</h4>
                    <p id="vehicles-status" class="text-yellow-400 text-xs mt-1">Fetching saved vehicles data...</p>
                    <div id="vehicles-data-display" class="data-snippet-box"></div>
                </div>

                <!-- Economy Data -->
                <div>
                    <h4 class="text-lg font-semibold text-gray-300">Economy Data:</h4>
                    <p id="economy-status" class="text-yellow-400 text-xs mt-1">Fetching saved economy data...</p>
                    <div id="economy-data-display" class="data-snippet-box"></div>
                </div>
            </div>
        </div>
        
        <p class="text-gray-400 text-sm mt-4 text-center">Note: Due to Crossplay limitations, this data reflects the last time the server saved, not real-time state.</p>
        
    </div>


    <script>
        // Configuration - Proxy URL
        const PROXY_URL = 'https://fs22-proxy.onrender.com';
        
        // Element references
        const WIDGET_ELEMENT = document.getElementById('fs22-status-widget');
        const MAP_IMAGE = document.getElementById('map-image');
        const MAP_LOADING = document.getElementById('map-loading');
        
        // Saved Data Display references
        const CAREER_STATUS = document.getElementById('career-status');
        const CAREER_DISPLAY = document.getElementById('career-data-display');
        const VEHICLES_STATUS = document.getElementById('vehicles-status');
        const VEHICLES_DISPLAY = document.getElementById('vehicles-data-display');
        const ECONOMY_STATUS = document.getElementById('economy-status');
        const ECONOMY_DISPLAY = document.getElementById('economy-data-display');


        /**
         * Generic fetcher function with error checking.
         * @param {string} endpoint The proxy endpoint (e.g., '/status').
         * @returns {Promise<any>} The raw XML/HTML text.
         */
        async function fetchData(endpoint) {
            const response = await fetch(PROXY_URL + endpoint);
            
            if (!response.ok) {
                let errorDetail = `Status ${response.status}`;
                try {
                    // Try to read a detailed JSON error response from the proxy
                    const errorJson = await response.json();
                    errorDetail = errorJson.details || errorJson.message || errorDetail;
                } catch (e) { 
                    // Ignore non-JSON errors
                } 
                throw new Error(`Request failed. ${errorDetail}`);
            }

            const text = await response.text();
            if (!text.trim().startsWith('<')) {
                throw new Error(`Invalid response format on ${endpoint}.`);
            }
            return text;
        }

        /**
         * Fetches all data and updates the dashboard.
         */
        async function updateDashboard() {
            try {
                // 1. Fetch Status XML (Proxy Endpoint: /status)
                const statusXml = await fetchData('/status');
                parseStatusAndDisplay(statusXml);

                // 2. Fetch Map Image (Proxy Endpoint: /mapimage)
                MAP_IMAGE.src = PROXY_URL + '/mapimage';
                MAP_LOADING.style.display = 'none';

                // 3. Fetch all Saved Data (Proxy Endpoints: /saved/career, /saved/vehicles, /saved/economy)
                await fetchSavedDataSnippet('/saved/career', CAREER_STATUS, CAREER_DISPLAY, 'Career Data');
                await fetchSavedDataSnippet('/saved/vehicles', VEHICLES_STATUS, VEHICLES_DISPLAY, 'Vehicles Data');
                await fetchSavedDataSnippet('/saved/economy', ECONOMY_STATUS, ECONOMY_DISPLAY, 'Economy Data');

            } catch (error) {
                handleFailure(error);
            }
        }
        
        /**
         * Fetches and displays a snippet of saved game data.
         * @param {string} endpoint The proxy endpoint to fetch.
         * @param {HTMLElement} statusElement The status text element.
         * @param {HTMLElement} displayElement The box to show the data.
         * @param {string} title A descriptive title for the fetch.
         */
        async function fetchSavedDataSnippet(endpoint, statusElement, displayElement, title) {
            statusElement.textContent = `Fetching ${title} from proxy...`;
            try {
                // Fetch the HTML/XML string
                const rawHtmlString = await fetchData(endpoint);
                
                // Saved data is wrapped in HTML tags which must be stripped for clean display.
                const cleanData = rawHtmlString
                    .replace(/<!DOCTYPE html>.*<body>\s*/is, '')
                    .replace(/\s*<\/body>.*<\/html>$/is, '')     
                    .trim();

                statusElement.classList.remove('text-yellow-400', 'critical-error');
                statusElement.classList.add('text-green-500');
                statusElement.textContent = `${title} Fetched Successfully`;
                
                // Display only the first 300 characters as a snippet
                displayElement.textContent = cleanData.substring(0, 300) + '...';
                
            } catch (error) {
                statusElement.classList.remove('text-yellow-400');
                statusElement.classList.add('critical-error');
                statusElement.textContent = `${title} Failed (Check Proxy/Deployment)`;
                displayElement.textContent = `Error: ${error.message}`;
                console.error(`${title} Error:`, error);
            }
        }


        // --- HANDLERS ---

        function parseStatusAndDisplay(xmlString) {
            try {
                const xmlDoc = $.parseXML(xmlString);
                const $xml = $(xmlDoc);

                const $server = $xml.find('Server');
                const $slots = $xml.find('Slots');
                
                if (!$server.length || !$slots.length) throw new Error("Invalid Server XML structure.");
                
                const serverName = $server.attr('name') || 'Unknown Server';
                const mapName = $server.attr('mapName') || 'Unknown Map';
                const numUsed = $slots.attr('numUsed') || '0';
                const capacity = $slots.attr('capacity') || '0';
                
                const htmlContent = `
                    <h4><span class="status-dot"></span> ${serverName}</h4>
                    <p>Status: <strong>Online</strong> | Map: ${mapName}</p>
                    <p>Players: <strong>${numUsed} / ${capacity}</strong></p>
                `;

                WIDGET_ELEMENT.classList.add('online');
                WIDGET_ELEMENT.innerHTML = htmlContent;
            } catch (e) {
                WIDGET_ELEMENT.classList.remove('online');
                WIDGET_ELEMENT.innerHTML = `<h4><span class="status-dot"></span> FS22 Server</h4><p class="error-status">Status Check Failed (Port 8080).</p>`;
                console.error("Status Widget Error:", e);
            }
        }
        
        function handleFailure(error) {
            console.error("Dashboard Failure:", error);
            const errorMessage = error.message || 'Check proxy URL or server connection.';
            
            WIDGET_ELEMENT.classList.remove('online');
            WIDGET_ELEMENT.innerHTML = `
                <h4><span class="status-dot"></span> FS22 Server</h4>
                <p>Status: <strong>Offline / Proxy Error</strong></p>
                <p class="critical-error text-xs mt-2">Detail: ${errorMessage}</p>
            `;
            MAP_LOADING.textContent = "Error loading data. Check proxy or server connection.";
        }


        document.addEventListener('DOMContentLoaded', updateDashboard);
        setInterval(updateDashboard, 30000); 

    </script>
</body>
</html>
