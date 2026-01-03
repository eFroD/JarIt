<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Recipe Agent</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        :root {
            --primary: #0891b2;
            --primary-dark: #0e7490;
            --success: #16a34a;
            --error: #dc2626;
            --warning: #ea580c;
        }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
        }
        .loading-spinner {
            border: 3px solid rgba(8, 145, 178, 0.1);
            border-top-color: #0891b2;
            border-radius: 50%;
            width: 40px;
            height: 40px;
            animation: spin 1s linear infinite;
        }
        @keyframes spin {
            to { transform: rotate(360deg); }
        }
        .recipe-preview {
            max-height: 600px;
            overflow-y-auto;
        }
    </style>
</head>
<body class="bg-gray-50">
    <div id="app" class="min-h-screen bg-gray-50">
        <!-- Navigation -->
        <nav class="bg-white shadow-sm border-b border-gray-200">
            <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
                <div class="flex justify-between h-16">
                    <div class="flex items-center">
                        <h1 class="text-2xl font-bold text-cyan-600">🍳 Recipe Agent</h1>
                    </div>
                    <div class="flex items-center gap-4" id="nav-right">
                        <!-- Populated by JS -->
                    </div>
                </div>
            </div>
        </nav>

        <!-- Main Content -->
        <main class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
            <div id="content-container">
                <!-- Populated by JS based on auth state -->
            </div>
        </main>
    </div>

    <script>
        // ============ STATE MANAGEMENT ============
        const appState = {
            authToken: localStorage.getItem('authToken') || null,
            user: JSON.parse(localStorage.getItem('user') || 'null'),
            currentStep: 'login', // login, dashboard, extract, preview
            videoUrl: '',
            targetLanguage: 'english',
            extractedRecipe: null,
            suggestedRecipe: null,
            error: null,
            isLoading: false,
            apiKeys: [],
            mealieApiKey: null,
        };

        const API_BASE = 'http://localhost:8000/api/v1';

        // ============ API CLIENT ============
        const api = {
            async request(endpoint, options = {}) {
                const headers = {
                    'Content-Type': 'application/json',
                    ...options.headers,
                };

                if (appState.authToken) {
                    headers['Authorization'] = `Bearer ${appState.authToken}`;
                }

                try {
                    const response = await fetch(`${API_BASE}${endpoint}`, {
                        ...options,
                        headers,
                    });

                    if (response.status === 401) {
                        appState.authToken = null;
                        appState.user = null;
                        localStorage.removeItem('authToken');
                        localStorage.removeItem('user');
                        render();
                        return null;
                    }

                    if (!response.ok) {
                        const error = await response.json().catch(() => ({ message: response.statusText }));
                        throw new Error(error.detail?.[0]?.msg || error.message || 'Request failed');
                    }

                    return await response.json();
                } catch (err) {
                    console.error('API Error:', err);
                    appState.error = err.message;
                    render();
                    throw err;
                }
            },

            register: (email, username, password) =>
                api.request('/auth/register', {
                    method: 'POST',
                    body: JSON.stringify({ email, username, password }),
                }),

            login: async (username, password) => {
                const formData = new FormData();
                formData.append('username', username);
                formData.append('password', password);

                const response = await fetch(`${API_BASE}/auth/login`, {
                    method: 'POST',
                    body: formData,
                });

                if (!response.ok) {
                    throw new Error('Invalid credentials');
                }

                return await response.json();
            },

            getCurrentUser: () => api.request('/users/me'),
            listApiKeys: () => api.request('/users/me/api-keys'),
            createApiKey: (serviceName, apiKey, baseUrl) =>
                api.request('/users/me/api-keys', {
                    method: 'POST',
                    body: JSON.stringify({ service_name: serviceName, api_key: apiKey, base_url: baseUrl }),
                }),

            deleteApiKey: (serviceName) =>
                api.request(`/users/me/api-keys/${serviceName}`, { method: 'DELETE' }),

            extractRecipe: (url, targetLanguage = 'english') =>
                api.request('/recipes/extract-recipe', {
                    method: 'POST',
                    body: JSON.stringify({ url, target_language: targetLanguage }),
                }),

            uploadToMealie: (recipe) =>
                api.request('/integrations/upload-mealie', {
                    method: 'POST',
                    body: JSON.stringify(recipe),
                }),

            verifyMealieUser: () => api.request('/integrations/verify-mealie-user'),
        };

        // ============ HANDLERS ============
        const handlers = {
            async register(e) {
                e.preventDefault();
                const form = e.target;
                const email = form.email.value;
                const username = form.username.value;
                const password = form.password.value;
                const confirmPassword = form.confirmPassword.value;

                appState.error = null;

                if (password !== confirmPassword) {
                    appState.error = 'Passwords do not match';
                    render();
                    return;
                }

                try {
                    appState.isLoading = true;
                    render();
                    await api.register(email, username, password);
                    appState.currentStep = 'login';
                    appState.error = null;
                    render();
                } catch (err) {
                    appState.error = err.message;
                    render();
                } finally {
                    appState.isLoading = false;
                }
            },

            async login(e) {
                e.preventDefault();
                const form = e.target;
                const username = form.username.value;
                const password = form.password.value;

                appState.error = null;

                try {
                    appState.isLoading = true;
                    render();
                    const data = await api.login(username, password);
                    appState.authToken = data.access_token;
                    localStorage.setItem('authToken', data.access_token);

                    const user = await api.getCurrentUser();
                    appState.user = user;
                    localStorage.setItem('user', JSON.stringify(user));

                    const apiKeys = await api.listApiKeys();
                    appState.apiKeys = apiKeys;
                    appState.mealieApiKey = apiKeys.find(k => k.service_name === 'mealie');

                    appState.currentStep = 'dashboard';
                    appState.error = null;
                    render();
                } catch (err) {
                    appState.error = err.message;
                    render();
                } finally {
                    appState.isLoading = false;
                }
            },

            logout() {
                appState.authToken = null;
                appState.user = null;
                appState.currentStep = 'login';
                localStorage.removeItem('authToken');
                localStorage.removeItem('user');
                render();
            },

            async handleExtractRecipe(e) {
                e.preventDefault();
                appState.error = null;

                const form = e.target;
                const url = form.videoUrl.value;

                if (!url) {
                    appState.error = 'Please enter a video URL';
                    render();
                    return;
                }

                try {
                    appState.isLoading = true;
                    appState.currentStep = 'extracting';
                    render();

                    const result = await api.extractRecipe(url, appState.targetLanguage);
                    appState.extractedRecipe = result.recipe;
                    appState.suggestedRecipe = result.suggested_version;
                    appState.error = result.error_info?.error || null;
                    appState.currentStep = 'preview';
                    render();
                } catch (err) {
                    appState.error = err.message;
                    appState.currentStep = 'dashboard';
                    render();
                } finally {
                    appState.isLoading = false;
                }
            },

            async uploadRecipe() {
                appState.error = null;

                if (!appState.extractedRecipe) {
                    appState.error = 'No recipe to upload';
                    render();
                    return;
                }

                try {
                    appState.isLoading = true;
                    render();

                    await api.uploadToMealie(appState.extractedRecipe);
                    appState.error = null;
                    alert('Recipe uploaded to Mealie successfully!');
                    appState.currentStep = 'dashboard';
                    appState.extractedRecipe = null;
                    appState.suggestedRecipe = null;
                    appState.videoUrl = '';
                    render();
                } catch (err) {
                    appState.error = `Upload failed: ${err.message}`;
                    render();
                } finally {
                    appState.isLoading = false;
                }
            },

            async saveMealieKey(e) {
                e.preventDefault();
                const form = e.target;
                const apiKey = form.mealieApiKey.value;
                const baseUrl = form.mealieBaseUrl.value;

                appState.error = null;

                try {
                    appState.isLoading = true;
                    render();

                    await api.createApiKey('mealie', apiKey, baseUrl);
                    appState.mealieApiKey = { service_name: 'mealie', api_key: apiKey, base_url: baseUrl };

                    const apiKeys = await api.listApiKeys();
                    appState.apiKeys = apiKeys;

                    appState.error = null;
                    render();
                } catch (err) {
                    appState.error = err.message;
                    render();
                } finally {
                    appState.isLoading = false;
                }
            },

            editRecipeField(field, value) {
                if (appState.extractedRecipe) {
                    appState.extractedRecipe[field] = value;
                    render();
                }
            },

            goBack() {
                appState.currentStep = 'dashboard';
                appState.extractedRecipe = null;
                appState.suggestedRecipe = null;
                appState.videoUrl = '';
                appState.error = null;
                render();
            },
        };

        // ============ RENDER FUNCTIONS ============
        function renderLoginPage() {
            return `
                <div class="min-h-[calc(100vh-100px)] flex items-center justify-center">
                    <div class="w-full max-w-md">
                        <div class="bg-white rounded-lg shadow-md p-8">
                            <h2 class="text-2xl font-bold text-gray-900 mb-6">Sign In</h2>
                            <form id="login-form" class="space-y-4">
                                <div>
                                    <label class="block text-sm font-medium text-gray-700 mb-1">Username</label>
                                    <input type="text" name="username" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                                </div>
                                <div>
                                    <label class="block text-sm font-medium text-gray-700 mb-1">Password</label>
                                    <input type="password" name="password" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                                </div>
                                <button type="submit" class="w-full bg-cyan-600 text-white py-2 rounded-lg font-medium hover:bg-cyan-700 transition" ${appState.isLoading ? 'disabled' : ''}>
                                    ${appState.isLoading ? 'Signing In...' : 'Sign In'}
                                </button>
                            </form>
                            <p class="text-center text-sm text-gray-600 mt-4">
                                Don't have an account? <button onclick="appState.currentStep = 'register'; render();" class="text-cyan-600 hover:text-cyan-700 font-medium">Register</button>
                            </p>
                        </div>
                    </div>
                </div>
            `;
        }

        function renderRegisterPage() {
            return `
                <div class="min-h-[calc(100vh-100px)] flex items-center justify-center">
                    <div class="w-full max-w-md">
                        <div class="bg-white rounded-lg shadow-md p-8">
                            <h2 class="text-2xl font-bold text-gray-900 mb-6">Create Account</h2>
                            <form id="register-form" class="space-y-4">
                                <div>
                                    <label class="block text-sm font-medium text-gray-700 mb-1">Email</label>
                                    <input type="email" name="email" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                                </div>
                                <div>
                                    <label class="block text-sm font-medium text-gray-700 mb-1">Username (3-50 chars)</label>
                                    <input type="text" name="username" minlength="3" maxlength="50" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                                </div>
                                <div>
                                    <label class="block text-sm font-medium text-gray-700 mb-1">Password (min 8 chars)</label>
                                    <input type="password" name="password" minlength="8" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                                </div>
                                <div>
                                    <label class="block text-sm font-medium text-gray-700 mb-1">Confirm Password</label>
                                    <input type="password" name="confirmPassword" minlength="8" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                                </div>
                                <button type="submit" class="w-full bg-cyan-600 text-white py-2 rounded-lg font-medium hover:bg-cyan-700 transition" ${appState.isLoading ? 'disabled' : ''}>
                                    ${appState.isLoading ? 'Creating Account...' : 'Register'}
                                </button>
                            </form>
                            <p class="text-center text-sm text-gray-600 mt-4">
                                Already have an account? <button onclick="appState.currentStep = 'login'; render();" class="text-cyan-600 hover:text-cyan-700 font-medium">Sign In</button>
                            </p>
                        </div>
                    </div>
                </div>
            `;
        }

        function renderDashboard() {
            return `
                <div class="space-y-8">
                    <!-- Extract Recipe Section -->
                    <div class="bg-white rounded-lg shadow-md p-8">
                        <h2 class="text-2xl font-bold text-gray-900 mb-6">Extract Recipe from Video</h2>
                        <form id="extract-form" class="space-y-4">
                            <div>
                                <label class="block text-sm font-medium text-gray-700 mb-1">Video URL</label>
                                <input type="url" name="videoUrl" value="${appState.videoUrl}" placeholder="https://www.youtube.com/watch?v=..." required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                            </div>
                            <div>
                                <label class="block text-sm font-medium text-gray-700 mb-1">Target Language</label>
                                <select name="targetLanguage" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent">
                                    <option value="english">English</option>
                                    <option value="german">German</option>
                                    <option value="spanish">Spanish</option>
                                    <option value="french">French</option>
                                </select>
                            </div>
                            <button type="submit" class="w-full bg-cyan-600 text-white py-2 rounded-lg font-medium hover:bg-cyan-700 transition" ${appState.isLoading ? 'disabled' : ''}>
                                ${appState.isLoading ? 'Extracting...' : 'Extract Recipe'}
                            </button>
                        </form>
                    </div>

                    <!-- Mealie API Key Section -->
                    <div class="bg-white rounded-lg shadow-md p-8">
                        <h2 class="text-2xl font-bold text-gray-900 mb-6">Mealie Configuration</h2>
                        ${appState.mealieApiKey ? `
                            <div class="bg-green-50 border border-green-200 rounded-lg p-4 mb-4">
                                <p class="text-green-800"><strong>✓ Mealie API Key Configured</strong></p>
                                <p class="text-green-700 text-sm">Base URL: ${appState.mealieApiKey.base_url || 'Not set'}</p>
                            </div>
                        ` : ''}
                        <form id="mealie-form" class="space-y-4">
                            <div>
                                <label class="block text-sm font-medium text-gray-700 mb-1">Mealie Base URL</label>
                                <input type="url" name="mealieBaseUrl" value="${appState.mealieApiKey?.base_url || ''}" placeholder="https://mealie.example.com" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                            </div>
                            <div>
                                <label class="block text-sm font-medium text-gray-700 mb-1">Mealie API Key</label>
                                <input type="password" name="mealieApiKey" placeholder="Enter your Mealie API key" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                            </div>
                            <button type="submit" class="w-full bg-cyan-600 text-white py-2 rounded-lg font-medium hover:bg-cyan-700 transition" ${appState.isLoading ? 'disabled' : ''}>
                                ${appState.isLoading ? 'Saving...' : 'Save Mealie Settings'}
                            </button>
                        </form>
                    </div>
                </div>
            `;
        }

        function renderExtractingPage() {
            return `
                <div class="flex flex-col items-center justify-center min-h-[calc(100vh-100px)]">
                    <div class="text-center">
                        <div class="loading-spinner mx-auto mb-4"></div>
                        <h2 class="text-2xl font-bold text-gray-900 mb-2">Extracting Recipe...</h2>
                        <p class="text-gray-600">This may take a moment. Please wait.</p>
                    </div>
                </div>
            `;
        }

        function renderPreviewPage() {
            if (!appState.extractedRecipe) {
                return renderDashboard();
            }

            const recipe = appState.extractedRecipe;

            return `
                <div class="space-y-6">
                    <div class="flex justify-between items-center">
                        <h2 class="text-2xl font-bold text-gray-900">Review Recipe</h2>
                        <button onclick="handlers.goBack();" class="px-4 py-2 bg-gray-300 text-gray-900 rounded-lg hover:bg-gray-400 transition">← Back</button>
                    </div>

                    ${appState.error ? `
                        <div class="bg-yellow-50 border border-yellow-200 rounded-lg p-4">
                            <p class="text-yellow-800"><strong>⚠ Warning:</strong> ${appState.error}</p>
                        </div>
                    ` : ''}

                    <div class="grid grid-cols-3 gap-6">
                        <!-- Recipe Edit Form -->
                        <div class="col-span-2 bg-white rounded-lg shadow-md p-8 space-y-4">
                            <div>
                                <label class="block text-sm font-medium text-gray-700 mb-1">Recipe Name</label>
                                <input type="text" value="${recipe.name || ''}" onchange="handlers.editRecipeField('name', this.value)" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                            </div>

                            <div>
                                <label class="block text-sm font-medium text-gray-700 mb-1">Description</label>
                                <textarea onchange="handlers.editRecipeField('description', this.value)" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" rows="3">${recipe.description || ''}</textarea>
                            </div>

                            <div class="grid grid-cols-3 gap-4">
                                <div>
                                    <label class="block text-sm font-medium text-gray-700 mb-1">Prep Time</label>
                                    <input type="text" value="${recipe.prepTime || ''}" onchange="handlers.editRecipeField('prepTime', this.value)" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" placeholder="PT15M" />
                                </div>
                                <div>
                                    <label class="block text-sm font-medium text-gray-700 mb-1">Cook Time</label>
                                    <input type="text" value="${recipe.cookTime || ''}" onchange="handlers.editRecipeField('cookTime', this.value)" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" placeholder="PT30M" />
                                </div>
                                <div>
                                    <label class="block text-sm font-medium text-gray-700 mb-1">Yield</label>
                                    <input type="text" value="${recipe.recipeYield || ''}" onchange="handlers.editRecipeField('recipeYield', this.value)" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" placeholder="4 servings" />
                                </div>
                            </div>

                            <div>
                                <label class="block text-sm font-medium text-gray-700 mb-1">Ingredients</label>
                                <div class="space-y-2 max-h-64 overflow-y-auto">
                                    ${recipe.recipeIngredient?.map((ing, i) => `
                                        <input type="text" value="${ing}" onchange="handlers.editRecipeField('recipeIngredient', [...appState.extractedRecipe.recipeIngredient.slice(0, ${i}), this.value, ...appState.extractedRecipe.recipeIngredient.slice(${i + 1})])" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-cyan-500 focus:border-transparent" />
                                    `).join('') || '<p class="text-gray-500">No ingredients</p>'}
                                </div>
                            </div>

                            <div class="flex gap-4">
                                <button onclick="handlers.uploadRecipe();" class="flex-1 bg-green-600 text-white py-2 rounded-lg font-medium hover:bg-green-700 transition" ${appState.isLoading ? 'disabled' : ''}>
                                    ${appState.isLoading ? 'Uploading...' : '✓ Approve & Upload to Mealie'}
                                </button>
                                <button onclick="handlers.goBack();" class="flex-1 bg-gray-300 text-gray-900 py-2 rounded-lg font-medium hover:bg-gray-400 transition">
                                    ✗ Cancel
                                </button>
                            </div>
                        </div>

                        <!-- Preview Sidebar -->
                        <div class="bg-white rounded-lg shadow-md p-6 h-fit recipe-preview">
                            <h3 class="font-bold text-lg mb-4 text-gray-900">${recipe.name || 'Untitled Recipe'}</h3>
                            ${recipe.image ? `<img src="${recipe.image}" alt="${recipe.name}" class="w-full rounded-lg mb-4 object-cover h-40" />` : ''}
                            <div class="space-y-3 text-sm">
                                <div>
                                    <p class="font-semibold text-gray-700">Category</p>
                                    <p class="text-gray-600">${recipe.recipeCategory || 'N/A'}</p>
                                </div>
                                <div>
                                    <p class="font-semibold text-gray-700">Cuisine</p>
                                    <p class="text-gray-600">${recipe.recipeCuisine || 'N/A'}</p>
                                </div>
                                <div>
                                    <p class="font-semibold text-gray-700">Yield</p>
                                    <p class="text-gray-600">${recipe.recipeYield || 'N/A'}</p>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            `;
        }

        function renderNavRight() {
            if (!appState.authToken) return '';

            return `
                <div class="flex items-center gap-4">
                    <span class="text-gray-700">Welcome, <strong>${appState.user?.username}</strong></span>
                    <button onclick="handlers.logout();" class="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700 transition font-medium">
                        Logout
                    </button>
                </div>
            `;
        }

        // ============ MAIN RENDER ============
        function render() {
            const navRight = document.getElementById('nav-right');
            navRight.innerHTML = renderNavRight();

            const container = document.getElementById('content-container');

            if (appState.error && appState.currentStep !== 'preview') {
                container.innerHTML = `
                    <div class="bg-red-50 border border-red-200 rounded-lg p-4 mb-4">
                        <div class="flex justify-between items-center">
                            <p class="text-red-800"><strong>✗ Error:</strong> ${appState.error}</p>
                            <button onclick="appState.error = null; render();" class="text-red-600 hover:text-red-800">×</button>
                        </div>
                    </div>
                ` + getPageContent();
            } else {
                container.innerHTML = getPageContent();
            }

            // Attach event listeners
            if (appState.currentStep === 'login') {
                document.getElementById('login-form')?.addEventListener('submit', handlers.login);
            } else if (appState.currentStep === 'register') {
                document.getElementById('register-form')?.addEventListener('submit', handlers.register);
            } else if (appState.currentStep === 'dashboard') {
                document.getElementById('extract-form')?.addEventListener('submit', handlers.handleExtractRecipe);
                document.getElementById('mealie-form')?.addEventListener('submit', handlers.saveMealieKey);
            }
        }

        function getPageContent() {
            if (!appState.authToken) {
                return appState.currentStep === 'register' ? renderRegisterPage() : renderLoginPage();
            }

            switch (appState.currentStep) {
                case 'extracting':
                    return renderExtractingPage();
                case 'preview':
                    return renderPreviewPage();
                default:
                    return renderDashboard();
            }
        }

        // ============ INITIALIZATION ============
        async function initialize() {
            if (appState.authToken && appState.user) {
                appState.currentStep = 'dashboard';
                try {
                    const apiKeys = await api.listApiKeys();
                    appState.apiKeys = apiKeys;
                    appState.mealieApiKey = apiKeys.find(k => k.service_name === 'mealie');
                } catch (err) {
                    console.error('Failed to load API keys:', err);
                }
            } else {
                appState.currentStep = 'login';
            }
            render();
        }

        // Start the app
        initialize();
    </script>
</body>
</html>