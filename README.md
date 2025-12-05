<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FS22 Server Dashboard</title>
    <!-- Load Tailwind CSS via CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
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
        .vehicle-icon {
            position: absolute;
            /* Placeholder size for equipment dots */
            width: 10px;
            height: 10px;
            background-color: rgb(255, 0, 0); /* Red dot for vehicle */
            border-radius: 50%;
            transform: translate(-50%, -50%); /* Center the dot on the coordinates */
            box-shadow: 0 0 5px rgba(255, 0, 0, 0.7);
            cursor: pointer;
            z-index: 10;
        }
        .vehicle-icon:hover {
            box-shadow: 0 0 10px rgba(255, 255, 0, 1);
        }
        .tooltip {
            position: absolute;
            background: rgba(0, 0, 0, 0.8);
            color: white;
            padding: 5px 10px;
            border-radius: 4px;
            font-size: 0.8rem;
            white-space: nowrap;
            z-index: 20;
            pointer-events: none; /* Allows clicks to pass through */
        }
        .field-label {
            position: absolute;
            background: rgba(0, 0, 0, 0.6);
            color: white;
            padding: 2px 5px;
            border-radius: 3px;
            font-size: 0.7rem;
            transform: translate(-50%, -50%);
            z-index: 5;
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
                    <!-- Vehicle icons will be dynamically added here -->
                    <!-- Field labels will be dynamically added here -->
                    <p id="map-loading" class="absolute inset-0 flex items-center justify-center text-white bg-gray-900/50">Loading map data...</p>
                </div>
            </div>

            <!-- Field Status and Info (Sidebar) -->
            <div class="lg:col-span-1 p-4 bg-gray-800 rounded-xl shadow-xl">
                <h3 class="text-xl font-bold text-gray-100 mb-4">Game & Field Status</h3>
                <div id="field-status-list" class="space-y-3 text-sm text-gray-300">
                    <p class="loading-status">Fetching game time and fields...</p>
                </div>
            </div>
        </div>
    </div>


    <script>
        // Configuration
        const PROXY_URL = 'https://fs22-data-proxy.onrender.com';
        const WIDGET_ELEMENT = document.getElementById('fs22-status-widget');
        const MAP_CONTAINER = document.getElementById('map-container');
        const MAP_IMAGE = document.getElementById('map-image');
        const FIELD_LIST = document.getElementById('field-status-list');
        const MAP_LOADING = document.getElementById('map-loading');
        
        // Map constants (approximate values for Big Flats Texas map)
        // These are often needed to translate game coordinates (X, Z) to screen pixels (left, top)
        const MAP_MIN_X = -1024;
        const MAP_MAX_X = 1024;
        const MAP_MIN_Z = -1024;
        const MAP_MAX_Z = 1024;
        const MAP_SIZE = MAP_MAX_X - MAP_MIN_X; // 2048

        /**
         * Fetches all data and updates the dashboard.
         */
        async function updateDashboard() {
            try {
                // 1. Fetch Status (for name, players)
                const statusResponse = await fetch(PROXY_URL + '/status');
                const statusXml = await statusResponse.text();

                // 2. Fetch Career (for game time, field data)
                const careerResponse = await fetch(PROXY_URL + '/career');
                const careerXml = await careerResponse.text();

                // 3. Fetch Vehicles (for equipment locations)
                const vehiclesResponse = await fetch(PROXY_URL + '/vehicles');
                const vehiclesXml = await vehiclesResponse.text();

                // 4. Fetch Map Image (The raw JPG)
                MAP_IMAGE.src = PROXY_URL + '/mapimage';
                MAP_LOADING.style.display = 'none'; // Hide loading text once image URL is set

                // Wait for the image to load to get dimensions for coordinates
                await new Promise((resolve, reject) => {
                    MAP_IMAGE.onload = resolve;
                    MAP_IMAGE.onerror = reject;
                });
                
                // --- Parsing and Display ---
                
                // Parse Status and Display (Top Widget)
                parseStatusAndDisplay(statusXml);
                
                // Parse Career and Display (Field Status)
                parseCareerAndDisplay(careerXml);

                // Parse Vehicles and Display (Map Markers)
                parseVehiclesAndDisplay(vehiclesXml);

            } catch (error) {
                console.error("Dashboard Update Failed:", error);
                handleFailure(error);
            }
        }

        // --- PART 1: STATUS WIDGET (Player Count, Name) ---

        function parseStatusAndDisplay(xmlString) {
            const parser = new DOMParser();
            const xmlDoc = parser.parseFromString(xmlString, "text/xml");
            
            const serverElement = xmlDoc.getElementsByTagName('Server')[0];
            const slotsElement = xmlDoc.getElementsByTagName('Slots')[0];

            if (!serverElement || !slotsElement) {
                WIDGET_ELEMENT.innerHTML = '<p class="error-status">Error: Invalid server response structure.</p>';
                return;
            }

            const serverName = serverElement.getAttribute('name') || 'Unknown Server';
            const mapName = serverElement.getAttribute('mapName') || 'Unknown Map';
            const numUsed = slotsElement.getAttribute('numUsed') || '0';
            const capacity = slotsElement.getAttribute('capacity') || '0';

            const htmlContent = `
                <h4><span class="status-dot"></span> ${serverName}</h4>
                <p>Status: <strong>Online</strong> | Map: ${mapName}</p>
                <p>Players: <strong>${numUsed} / ${capacity}</strong></p>
            `;

            WIDGET_ELEMENT.classList.add('online');
            WIDGET_ELEMENT.innerHTML = htmlContent;
        }

        // --- PART 2: FIELD STATUS (Game Time, Fields) ---
        
        function formatGameTime(dayTime) {
            const totalSeconds = parseInt(dayTime, 10);
            const hours = Math.floor(totalSeconds / 3600);
            const minutes = Math.floor((totalSeconds % 3600) / 60);
            const seconds = totalSeconds % 60;
            return `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;
        }

        function parseCareerAndDisplay(xmlString) {
            const parser = new DOMParser();
            const xmlDoc = parser.parseFromString(xmlString, "text/xml");
            
            const serverElement = xmlDoc.getElementsByTagName('Server')[0];
            const fieldsElements = xmlDoc.getElementsByTagName('Field');
            
            let htmlContent = '';
            
            // 1. Display Game Time
            const dayTime = serverElement.getAttribute('dayTime') || '0';
            const formattedTime = formatGameTime(dayTime);

            htmlContent += `<p class="text-lg font-semibold text-white">Current Game Time: <span class="text-yellow-400">${formattedTime}</span></p><hr class="my-3 border-gray-700">`;

            // 2. Display Field Status (using a simple ownership/ID view)
            htmlContent += '<p class="font-bold text-gray-200 mb-2">Field Ownership & IDs:</p>';
            
            Array.from(fieldsElements).forEach(field => {
                const id = field.getAttribute('id');
                const isOwned = field.getAttribute('isOwned') === 'true';
                const x = parseFloat(field.getAttribute('x'));
                const z = parseFloat(field.getAttribute('z'));
                
                const statusText = isOwned ? 'Owned' : 'Available';
                const statusClass = isOwned ? 'text-green-400' : 'text-red-400';

                htmlContent += `<p class="flex justify-between items-center">
                    Field #${id} <span class="${statusClass} font-medium">${statusText}</span>
                </p>`;
                
                // Add field markers to the map
                addMapMarker(MAP_CONTAINER, x, z, `Field ${id}`, 'field');
            });

            FIELD_LIST.innerHTML = htmlContent;
        }

        // --- PART 3: VEHICLE LOCATIONS (Map Markers) ---

        function parseVehiclesAndDisplay(xmlString) {
            // Remove previous vehicle icons
            document.querySelectorAll('.vehicle-icon').forEach(el => el.remove());
            
            const parser = new DOMParser();
            const xmlDoc = parser.parseFromString(xmlString, "text/xml");
            const vehicleElements = xmlDoc.getElementsByTagName('Vehicle');

            Array.from(vehicleElements).forEach(vehicle => {
                const name = vehicle.getAttribute('name');
                const category = vehicle.getAttribute('category');
                // FS uses X and Z for horizontal position, Y for vertical. We use X and Z.
                const x = parseFloat(vehicle.getAttribute('x'));
                const z = parseFloat(vehicle.getAttribute('z')); 
                const fillLevels = vehicle.getAttribute('fillLevels') || '0';
                
                let tooltipText = `${name} (${category})`;
                
                if (fillLevels !== '0') {
                    // Simple check for fuel/seed levels
                    tooltipText += ` | Fill: ${fillLevels.split(' ')[0]}%`; 
                }

                // Add vehicle icon to the map container
                addMapMarker(MAP_CONTAINER, x, z, tooltipText, 'vehicle');
            });
        }
        
        // --- HELPER: Coordinate Conversion and Marker Creation ---

        /**
         * Converts game coordinates to percentage coordinates relative to the map image.
         * @param {number} x Game X coordinate.
         * @param {number} z Game Z coordinate.
         * @returns {{left: string, top: string}} Percentage positions.
         */
        function gameCoordsToPercent(x, z) {
            // Normalize X and Z coordinates based on the map's bounds (e.g., -1024 to 1024)
            // (Coord - Min) / Size = 0 to 1 ratio
            const percentX = (x - MAP_MIN_X) / MAP_SIZE;
            const percentZ = (z - MAP_MIN_Z) / MAP_SIZE;

            // X corresponds to CSS 'left', Z corresponds to CSS 'top' (since positive Z is 'down' on the map)
            // Need to invert Z if the map image is oriented North-up
            // For now, assume standard orientation (X = left, Z = top)
            return {
                left: `${percentX * 100}%`,
                top: `${(1 - percentZ) * 100}%` // Invert Z to match map visual orientation
            };
        }

        /**
         * Creates and places a marker on the map container.
         */
        function addMapMarker(container, x, z, label, type) {
            if (isNaN(x) || isNaN(z)) return;

            const pos = gameCoordsToPercent(x, z);
            
            const marker = document.createElement('div');
            marker.style.left = pos.left;
            marker.style.top = pos.top;
            
            if (type === 'vehicle') {
                // Vehicle Icon (Red Dot)
                marker.className = 'vehicle-icon';
                
                // Create and attach tooltip
                const tooltip = document.createElement('span');
                tooltip.className = 'tooltip hidden';
                tooltip.textContent = label;
                marker.appendChild(tooltip);

                marker.onmouseover = () => tooltip.classList.remove('hidden');
                marker.onmouseout = () => tooltip.classList.add('hidden');
                
            } else if (type === 'field') {
                // Field Label (ID Label)
                marker.className = 'field-label';
                marker.textContent = label;
                marker.style.backgroundColor = 'rgba(255, 255, 255, 0.4)';
                marker.style.color = '#1f2937';
                marker.style.fontWeight = 'bold';
            }
            
            container.appendChild(marker);
        }

        // --- GLOBAL HANDLERS ---

        function handleFailure(error) {
            console.error("Dashboard Failure:", error);
            const errorMessage = error.message || 'Check proxy URL or server firewall.';
            WIDGET_ELEMENT.classList.remove('online');
            WIDGET_ELEMENT.innerHTML = `
                <h4><span class="status-dot"></span> FS22 Server</h4>
                <p>Status: <strong>Offline</strong></p>
                <p class="error-status">Could not load all data.</p>
                <p class="error-status text-xs mt-2">${errorMessage}</p>
            `;
            FIELD_LIST.innerHTML = `<p class="error-status">Failed to load game data.</p>`;
            MAP_LOADING.textContent = "Error loading map. Check proxy or server connection.";
        }


        // Start fetching status data when the page loads
        document.addEventListener('DOMContentLoaded', updateDashboard);

        // Refresh all data every 30 seconds
        setInterval(updateDashboard, 30000); 

    </script>
</body>
</html>
