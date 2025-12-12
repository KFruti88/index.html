<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FS22 Dashboard Preview</title>
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
            padding-top: 100%; /* 1:1 Aspect Ratio (for a square map) */
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
    </style>
</head>
<body class="p-4 sm:p-8 flex justify-center items-start min-h-screen">

    <div class="w-full max-w-4xl space-y-4">
        <!-- Live Status Widget (Top Bar) -->
        <div id="fs22-status-widget" class="shadow-xl">
            <p class="loading-status">Loading server status...</p>
        </div>

        <!-- Dashboard Content Grid -->
        <div class="grid grid-cols-1 lg:grid-cols-3 gap-4">
            <!-- Map Container (Large Area) -->
            <div class="lg:col-span-2 p-4 bg-gray-800 rounded-xl shadow-xl">
                <h3 class="text-xl font-bold text-gray-100 mb-2">Live Map & Equipment Locations</h3>
                <div id="map-container">
                    <img id="map-image" src="" alt="Loading Map..." class="animate-pulse">
                    <p id="map-loading" class="absolute inset-0 flex items-center justify-center text-white bg-gray-900/50">Loading map data...</p>
                </div>
            </div>

            <!-- Field Status and Info (Sidebar) -->
            <div class="lg:col-span-1 p-4 bg-gray-800 rounded-xl shadow-xl">
                <h3 class="text-xl font-bold text-gray-100 mb-4">Advanced Server Diagnostic (RCON)</h3>
                <div id="field-status-list" class="space-y-3 text-sm text-gray-300">
                    <p class="critical-error">External Data Access Unavailable.</p>
                    <p class="text-yellow-400 text-xs mt-2">Running RCON diagnostic check...</p>
                </div>
            </div>
        </div>
        
        <!-- DEBUG PANEL WITH ENDPOINT LINKS -->
        <div class="p-4 bg-gray-800 rounded-xl shadow-xl border border-red-500/50">
            <h3 class="text-xl font-bold text-red-400 mb-2">FS22 Server Debug Endpoints (Port 8080)</h3>
            <p class="text-sm text-gray-300 mb-2">
                Use the /rcon-test endpoint on the proxy URL below to get the definitive RCON status.
            </p>
            <div class="space-y-1 text-xs break-all text-gray-400">
                <a id="rcon-test-link" href="#" target="_blank" class="hover:text-red-300 block">
                    <span class="font-mono">Proxy RCON Diagnostic Test Link (Click me)</span>
                </a>
            </div>
        </div>
    </div>


    <script>
        // Configuration - Proxy URL is hardcoded here for the preview.
        const PROXY_URL = 'https://fs22-proxy.onrender.com';
        
        const WIDGET_ELEMENT = document.getElementById('fs22-status-widget');
        const MAP_IMAGE = document.getElementById('map-image');
        const FIELD_LIST = document.getElementById('field-status-list');
        const MAP_LOADING = document.getElementById('map-loading');
        const RCON_TEST_LINK = document.getElementById('rcon-test-link');

        // Set the RCON diagnostic link
        RCON_TEST_LINK.href = PROXY_URL + '/rcon-test';

        /**
         * Generic fetcher function with error checking.
         * @param {string} endpoint The proxy endpoint (e.g., '/status').
         * @param {boolean} isJson Set to true for JSON endpoints.
         * @returns {Promise<any>} The raw XML text or parsed JSON object.
         */
        async function fetchData(endpoint, isJson = false) {
            const response = await fetch(PROXY_URL + endpoint);
            
            if (!response.ok) {
                let errorDetail = `Status ${response.status} from proxy.`;
                try {
                    // Try to read a detailed JSON error response from the proxy
                    const errorJson = await response.json();
                    errorDetail = errorJson.details || errorJson.message || errorDetail;
                } catch (e) { 
                    // If parsing as JSON fails, just use the status code
                } 
                throw new Error(`Proxy error on ${endpoint}: Request failed. ${errorDetail}`);
            }

            if (isJson) {
                return await response.json();
            } else {
                const text = await response.text();
                // Basic XML check (since the dedicated-server-stats feed is XML)
                if (!text.trim().startsWith('<')) {
                    throw new Error(`Invalid response format on ${endpoint}. Not XML.`);
                }
                return text;
            }
        }

        /**
         * Fetches all data and updates the dashboard.
         */
        async function updateDashboard() {
            try {
                // 1. Fetch Status XML (Proxy Endpoint: /status -> dedicated-server-stats.xml)
                const statusXml = await fetchData('/status');
                parseStatusAndDisplay(statusXml);

                // 2. Fetch Map Image (Proxy Endpoint: /mapimage -> dedicated-server-stats-map.jpg)
                MAP_IMAGE.src = PROXY_URL + '/mapimage';
                MAP_LOADING.style.display = 'none';

                // 3. RCON Diagnostic Check (Proxy Endpoint: /career -> dedicated-server-savegame.html?file=careerSavegame)
                // This request triggers the RCON connection attempt on the server side.
                await checkRconDiagnostic();

            } catch (error) {
                handleFailure(error);
            }
        }
        
        /**
         * Checks the RCON diagnostic route (using the /career placeholder endpoint).
         */
        async function checkRconDiagnostic() {
             try {
                // We use /career as a trigger for the server-side RCON test
                const diagnosticJson = await fetchData('/career', true);
                
                // If it succeeds (miracle case)
                FIELD_LIST.innerHTML = '<p class="text-green-500 font-bold">SUCCESS: External RCON Access is Open!</p>';

            } catch (error) {
                const errorText = error.message || '';
                
                // If the error includes the specific RCON Timeout message, display the diagnostic failure
                if (errorText.includes('RCON Connection Timeout')) {
                    handleDiagnosticError({
                        message: "RCON Service Access Unavailable",
                        details: "Proxy could not connect to RCON Port 27016. If G-Portal confirmed the firewall is open, the RCON service inside the FS22 server is not running or is misconfigured."
                    });
                } else {
                    // Otherwise, show a generic failure
                    FIELD_LIST.innerHTML = `<p class="critical-error">Diagnostic Error: ${error.message}</p>`;
                }
                console.error("RCON Diagnostic Error:", error);
            }
        }


        // --- HANDLERS ---

        function parseStatusAndDisplay(xmlString) {
            try {
                // jQuery XML parsing is necessary here because the widget uses it
                const xmlDoc = $.parseXML(xmlString);
                const $xml = $(xmlDoc);

                const $server = $xml.find('Server');
                const $slots = $xml.find('Slots');
                
                if (!$server.length || !$slots.length) throw new Error("Invalid Server XML structure.");
                
                const serverName = $server.attr('name') || 'Unknown Server';
                const mapName = $server.attr('mapName') || 'Unknown Map';
                const numUsed = $slots.attr('numUsed') || '0';
                const capacity = $slots.attr('capacity') || '0';

                // FS22 XML Uptime is a huge number representing milliseconds; not friendly to display
                // We simplify the display here since the XML format is inconsistent
                
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
        
        function handleDiagnosticError(errorData) {
            const message = errorData.message || "Connection Failure";
            const details = errorData.details || "Details unavailable.";
            
            FIELD_LIST.innerHTML = `
                <p class="critical-error text-base">${message}</p>
                <hr class="my-2 border-gray-700">
                <p class="text-sm text-yellow-400">RCON Diagnostic Result:</p>
                <p class="text-xs break-words">${details}</p>
                <p class="text-sm mt-3">
                    <span class="critical-error">CONCLUSION:</span> The RCON service is not responding externally.
                </p>
            `;
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
            FIELD_LIST.innerHTML = `<p class="critical-error">Proxy/Connectivity Failure.</p>`;
            MAP_LOADING.textContent = "Error loading map. Check proxy or server connection.";
        }


        document.addEventListener('DOMContentLoaded', updateDashboard);
        setInterval(updateDashboard, 30000); 

    </script>
</body>
</html>
