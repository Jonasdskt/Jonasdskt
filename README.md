pip install Flask networkx folium

projeto_logistica/
│
├── static/
│   ├── style.css
│   └── leaflet.css
│
├── templates/
│   └── index.html
│
├── app.py
│
└── cidades.json

from flask import Flask, render_template, request, jsonify
import networkx as nx
import folium

app = Flask(__name__)

# Carregar dados de cidades a partir de um arquivo JSON
with open('cidades.json') as f:
    cidades = json.load(f)

# Criar um grafo ponderado
G = nx.Graph()

for cidade, dados in cidades.items():
    G.add_node(cidade, posicao=(dados['latitude'], dados['longitude']))

# Adicionar arestas ponderadas
arestas = [
    ("Foz do Iguaçu", "União da Vitória", {"weight": 0}),
    ("Joinville", "Chapecó", {"weight": 0}),
    # Adicione outras arestas conforme necessário
]

G.add_edges_from(arestas)

@app.route('/')
def index():
    return render_template('index.html', cidades=cidades)

@app.route('/calcular_rota', methods=['POST'])
def calcular_rota():
    source = request.form['source']
    target = request.form['target']

    shortest_path = nx.shortest_path(G, source=source, target=target, weight="weight")
    total_distance = nx.shortest_path_length(G, source=source, target=target, weight="weight")
    total_cost = total_distance * 20  # Custo por quilômetro rodado

    # Formatar resultados
    results = {
        "menor_caminho": shortest_path,
        "distancia_percorrida": total_distance,
        "custo_transporte": total_cost
    }

    return jsonify(results)

if __name__ == '__main__':
    app.run(debug=True)

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Logística</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
    <link rel="stylesheet" href="{{ url_for('static', filename='leaflet.css') }}">
</head>
<body>
    <h1>Logística</h1>
    <form id="calculo-form">
        <label for="source">Origem:</label>
        <select id="source" name="source">
            {% for cidade, dados in cidades.items() %}
                <option value="{{ cidade }}">{{ cidade }}</option>
            {% endfor %}
        </select>

        <label for="target">Destino:</label>
        <select id="target" name="target">
            {% for cidade, dados in cidades.items() %}
                <option value="{{ cidade }}">{{ cidade }}</option>
            {% endfor %}
        </select>

        <button type="button" onclick="calcularRota()">Calcular Rota</button>
    </form>

    <div id="map"></div>

    <script src="https://code.jquery.com/jquery-3.6.4.min.js"></script>
    <script src="{{ url_for('static', filename='leaflet.js') }}"></script>
    <script>
        function calcularRota() {
            var source = $("#source").val();
            var target = $("#target").val();

            $.post('/calcular_rota', { source: source, target: target }, function(data) {
                alert("Menor caminho: " + data.menor_caminho.join(' -> '));
                alert("Distância percorrida: " + data.distancia_percorrida + " km");
                alert("Custo de transporte: R$" + data.custo_transporte.toFixed(2));

                // Adicionar código para exibir o mapa com o caminho percorrido
                var map = L.map('map').setView([-25.4296, -49.2719], 6);
                L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

                var pathCoordinates = [];
                for (var i = 0; i < data.menor_caminho.length; i++) {
                    var cidade = data.menor_caminho[i];
                    var posicao = cidades[cidade].posicao;
                    pathCoordinates.push([posicao[0], posicao[1]]);
                }

                L.polyline(pathCoordinates, { color: 'blue' }).addTo(map);

            });
        }
    </script>
</body>
</html>
