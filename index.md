---
layout: null
---
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Anass Cherraqi | Cybersecurity Engineer & SOC Analyst</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
</head>
<body class="bg-gray-900 text-gray-300 font-sans antialiased leading-relaxed">

    <!-- Header Section -->
    <header class="bg-gray-800 shadow-lg py-12">
        <div class="max-w-5xl mx-auto px-6 flex flex-col md:flex-row items-center gap-8">
            <!-- Profile Image (Uses your GitHub Avatar) -->
            <img src="https://github.com/mracherr.png" alt="Anass Cherraqi" class="w-40 h-40 rounded-full border-4 border-emerald-500 shadow-xl">
            
            <div class="text-center md:text-left">
                <h1 class="text-4xl md:text-5xl font-bold text-white tracking-tight">Anass Cherraqi</h1>
                <p class="text-xl text-emerald-400 mt-2 font-medium">Analyste SOC Cybersécurité Défensive</p>
                <p class="text-gray-400 mt-2 max-w-2xl">Spécialisé en sécurité opérationnelle, surveillance SIEM, threat hunting et développement de règles de détection (MITRE ATT&CK).</p>
                
                <!-- Social Links -->
                <div class="mt-6 flex justify-center md:justify-start gap-4">
                    <a href="mailto:cherraqianass01@gmail.com" class="text-gray-400 hover:text-emerald-400 transition text-2xl"><i class="fas fa-envelope"></i></a>
                    <a href="https://github.com/mracherr" target="_blank" class="text-gray-400 hover:text-emerald-400 transition text-2xl"><i class="fab fa-github"></i></a>
                    <a href="https://linkedin.com/in/anass-cherraqi" target="_blank" class="text-gray-400 hover:text-emerald-400 transition text-2xl"><i class="fab fa-linkedin"></i></a>
                </div>
            </div>
        </div>
    </header>

    <!-- Main Content -->
    <main class="max-w-5xl mx-auto px-6 py-12 grid grid-cols-1 md:grid-cols-3 gap-12">
        
        <!-- Left Column: Skills & Education -->
        <div class="space-y-10">
            <!-- Skills -->
            <section>
                <h2 class="text-2xl font-semibold text-white border-b-2 border-emerald-500 pb-2 mb-4">Compétences</h2>
                <ul class="space-y-3 text-sm">
                    <li><strong class="text-emerald-400">SIEM & Détection:</strong> Splunk, Elastic / ELK Stack</li>
                    <li><strong class="text-emerald-400">Threat Hunting:</strong> MITRE ATT&CK, Cyber Kill Chain</li>
                    <li><strong class="text-emerald-400">Réseau & SOAR:</strong> Suricata, Snort, Wireshark</li>
                    <li><strong class="text-emerald-400">Systèmes & Scripting:</strong> Linux, Windows, Python, Bash</li>
                </ul>
            </section>

            <!-- Certifications -->
            <section>
                <h2 class="text-2xl font-semibold text-white border-b-2 border-emerald-500 pb-2 mb-4">Certifications</h2>
                <ul class="list-disc list-inside space-y-1 text-sm">
                    <li>Hack The Box CPTS, CSDA</li>
                    <li>Fortinet - NSE 1, NSE 2, NSE 3</li>
                    <li>CCNA 1 & 2</li>
                    <li>AWS Cloud Foundations</li>
                </ul>
            </section>
        </div>

        <!-- Right Column: Experience & Blog -->
        <div class="md:col-span-2 space-y-12">
            
            <!-- Experience -->
            <section>
                <h2 class="text-2xl font-semibold text-white border-b-2 border-emerald-500 pb-2 mb-6">Expériences Professionnelles</h2>
                
                <div class="space-y-8">
                    <!-- OCP -->
                    <div>
                        <div class="flex justify-between items-baseline mb-1">
                            <h3 class="text-xl font-bold text-white">Consultant Cybersécurité</h3>
                            <span class="text-emerald-400 text-sm font-medium">Mar 2026 - Jul 2026</span>
                        </div>
                        <p class="text-gray-400 mb-2 font-medium">OCP Group</p>
                        <ul class="list-disc list-inside space-y-1 text-sm text-gray-300">
                            <li>Audit d'environnement industriel (C2M2).</li>
                            <li>Conception de segmentation réseau sécurisée.</li>
                        </ul>
                    </div>

                    <!-- IMS Technology -->
                    <div>
                        <div class="flex justify-between items-baseline mb-1">
                            <h3 class="text-xl font-bold text-white">Analyste SOC</h3>
                            <span class="text-emerald-400 text-sm font-medium">Fév 2024 - Jul 2024</span>
                        </div>
                        <p class="text-gray-400 mb-2 font-medium">IMS Technology</p>
                        <ul class="list-disc list-inside space-y-1 text-sm text-gray-300">
                            <li>Surveillance et triage d'alertes via Splunk et ELK.</li>
                            <li>Développement de règles de détection (MITRE ATT&CK).</li>
                        </ul>
                    </div>
                </div>
            </section>

            <!-- Blog Archive -->
            <section>
                <h2 class="text-2xl font-semibold text-white border-b-2 border-emerald-500 pb-2 mb-6">Articles de Blog</h2>
                <div class="grid grid-cols-1 gap-4">
                    {% for post in site.posts %}
                    <a href="{{ post.url | relative_url }}" class="block bg-gray-800 p-5 rounded-lg border border-gray-700 hover:border-emerald-500 hover:bg-gray-750 transition shadow-sm">
                        <span class="text-emerald-400 text-sm font-medium block mb-1">{{ post.date | date: "%B %d, %Y" }}</span>
                        <h3 class="text-lg font-bold text-white">{{ post.title }}</h3>
                    </a>
                    {% endfor %}
                </div>
            </section>

        </div>
    </main>
</body>
</html>