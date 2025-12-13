<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FS22 Live Map Console Compatible</title>
    <!-- PRODUCTION CSS: Compiled using Tailwind CLI in GitHub Actions -->
    <link href="dist/output.css" rel="stylesheet">
    <style>
        /* Custom styles for map area and loading */
        #map-container {
            min-height: 400px;
            background-color: #e5e7eb;
            background-image: url('https://placehold.co/800x400/16a34a/ffffff?text=Placeholder%20Map%20Image');
            background-size: cover;
            background-position: center;
        }
        .loading-animation {
            border-top-color: #3b82f6;
            -webkit-animation: spinner 1.5s linear infinite;
            animation: spinner 1.5s linear infinite;
        }
        @-webkit-keyframes spinner { to { -webkit-transform: rotate(360deg); } }
        @keyframes spinner { to { transform: rotate(360deg); } }
    </style>
</head>
<body class="bg-gray-900 text-gray-100 font-sans">

    <div class="container mx-auto p-4 max-w-4xl">
        <header class="text-center mb-6">
            <h1 class="text-4xl font-bold text-yellow-500">FS22 Live Map Console Compatible</h1>
            <p class="text-gray-400">Viewing synchronized data from your dedicated server.</p>
        </header>

        <!-- Game Status Summary -->
        <div id="game-status-summary" class="bg-gray-700 p-4 rounded-lg shadow-inner mb-6 grid grid-cols-2 gap-4 text-center">
            <p><strong class="text-yellow-300">Farm Name:</strong> <span id="farm-name" class="text-white">Loading...</span></p>
            <p><strong class="text-yellow-300">Money:</strong> <span id="farm-money" class="text-white">Loading...</span></p>
        </div>

        <!-- Map Visualization Area (Placeholder) -->
        <div id="map-container" class="rounded-lg shadow-xl overflow-hidden mb-6 relative">
            <div id="map-status" class="absolute inset-0 flex items-center justify-center bg-gray-900/70 text-lg font-mono">
                <!-- Status text changes based on fetch state -->
            </div>
        </div>

        <!-- Data Display Area -->
        <div class="bg-gray-800 p-6 rounded-lg shadow-2xl">
            <h2 class="text-2xl font-semibold mb-4 text-white">Live Vehicle Status</h2>
            <div id="vehicle-list" class="space-y-4">
                <!-- Vehicle data will be injected here -->
            </div>
        </div>
    </div>

    <script>
        const vehicleList = document.getElementById('vehicle-list');
        const mapStatus = document.getElementById('map-status');
        const farmNameElement = document.getElementById('farm-name');
        const farmMoneyElement = document.getElementById('farm-money');
        
        // --- SYNCHRONIZED FILE PATHS ---
        // Using window.location.origin to ensure a valid, absolute URL to fix parsing errors.
        const BASE_URL = window.location.origin;
        const VEHICLE_FILE_PATH = BASE_URL + '/map-data/vehicles.xml';
        const CAREER_FILE_PATH = BASE_URL + '/map-data/careerSavegame.xml'; 

        /**
         * Utility function to safely format money.
         */
        const formatMoney = (amount) => {
            if (isNaN(amount)) return amount;
            return new Intl.NumberFormat('en-US', {
                style: 'currency',
                currency: 'USD',
                minimumFractionDigits: 0,
                maximumFractionDigits: 0
            }).format(amount);
        };

        /**
         * Parses the Career Savegame XML and updates the summary.
         * @param {Document} xmlDoc The parsed careerSavegame XML Document.
         */
        function displayGameStatus(xmlDoc) {
            const settings = xmlDoc.getElementsByTagName('settings')[0];
            const statistics = xmlDoc.getElementsByTagName('statistics')[0];
            
            let name = 'N/A';
            let money = 'N/A';

            if (settings) {
                const savegameName = settings.getElementsByTagName('savegameName')[0];
                if (savegameName) {
                    name = savegameName.textContent;
                }
            }

            if (statistics) {
                const moneyElement = statistics.getElementsByTagName('money')[0];
                if (moneyElement) {
                    money = formatMoney(parseFloat(moneyElement.textContent));
                }
            }
            
            farmNameElement.textContent = name;
            farmMoneyElement.textContent = money;
        }

        /**
         * Parses the Vehicle XML and updates the vehicle list.
         * @param {Document} xmlDoc The parsed vehicles XML Document.
         */
        function displayVehicleData(xmlDoc) {
            const vehicles = xmlDoc.getElementsByTagName('vehicle');
            
            if (vehicles.length === 0) {
                vehicleList.textContent = 'No vehicles found in XML data.';
                return;
            }

            let htmlContent = '';
            
            for (let i = 0; i < vehicles.length; i++) {
                const vehicle = vehicles[i];
                
                const modNameAttribute = vehicle.getAttribute('modName');
                const name = modNameAttribute ? modNameAttribute.split('_').pop() : 'Unknown Vehicle';
                const farmId = vehicle.getAttribute('farmId');
                
                const component = vehicle.getElementsByTagName('component')[0];
                const position = component ? component.getAttribute('position') : 'N/A';
                
                let fillStatus = 'Empty/N/A';
                const fillUnit = vehicle.getElementsByTagName('fillUnit')[0];
                if (fillUnit) {
                    const unit = fillUnit.getElementsByTagName('unit')[0];
                    if (unit) {
                        const fillType = unit.getAttribute('fillType');
                        const fillLevel = parseFloat(unit.getAttribute('fillLevel')).toFixed(1);
                        fillStatus = `${fillType}: ${fillLevel} L`;
                    }
                }

                const cardColor = farmId === '1' ? 'border-yellow-500' : 'border-gray-500';

                htmlContent += `
                    <div class="bg-gray-700 p-4 rounded-lg shadow-md border-l-4 ${cardColor}">
                        <div class="flex justify-between items-center">
                            <h3 class="text-xl font-bold text-yellow-300">${name} (ID: ${vehicle.getAttribute('id')})</h3>
                            <span class="text-sm text-gray-300">${farmId === '1' ? 'YOUR FARM' : 'NPC/EXTERNAL'}</span>
                        </div>
                        <p class="text-gray-300 mt-2">
                            <strong class="text-gray-400">Position (X Y Z):</strong> <span class="font-mono">${position}</span>
                        </p>
                        <p class="text-gray-300">
                            <strong class="text-gray-400">Fuel/Fill:</strong> <span class="font-mono">${fillStatus}</span>
                        </p>
                    </div>
                `;
            }

            vehicleList.innerHTML = htmlContent;
            mapStatus.textContent = 'Data Loaded Successfully';
        }

        /**
         * Fetches and parses a single XML file.
         * @param {string} url Path to the XML file.
         * @returns {Promise<Document>} The parsed XML Document.
         */
        async function fetchAndParseXml(url) {
            const response = await fetch(url);
            if (!response.ok) {
                // If fetching the file fails, display the exact path we tried to access
                throw new Error(`HTTP error! Status: ${response.status}. Attempted path: ${url}`);
            }
            const xmlText = await response.text();
            const parser = new DOMParser();
            const xmlDoc = parser.parseFromString(xmlText, "text/xml");
            return xmlDoc;
        }

        /**
         * Main function to load all data concurrently.
         */
        async function loadData() {
            mapStatus.innerHTML = '<div class="loading-animation w-12 h-12 rounded-full border-4 border-gray-700"></div><p class="mt-3">Fetching live data...</p>';
            
            try {
                // Fetch both essential files simultaneously
                const [vehiclesDoc, careerDoc] = await Promise.all([
                    fetchAndParseXml(VEHICLE_FILE_PATH),
                    fetchAndParseXml(CAREER_FILE_PATH)
                ]);

                // Display data from both files
                displayGameStatus(careerDoc);
                displayVehicleData(vehiclesDoc);

            } catch (error) {
                console.error("Error fetching or processing XML:", error);
                // Display the detailed error message to help troubleshooting.
                mapStatus.innerHTML = `<p class="text-red-400">ERROR: Could not load data.</p><p class="text-sm text-gray-400 mt-2">Error Detail: ${error.message}.</p>`;
                
                farmNameElement.textContent = 'ERROR';
                farmMoneyElement.textContent = 'ERROR';
            }
        }

        // Set up initial load and periodic reload
        window.onload = () => {
            loadData();
            // Reload every 10 seconds (10000 milliseconds)
            setInterval(loadData, 10000); 
        };
    </script>
</body>
</html>
