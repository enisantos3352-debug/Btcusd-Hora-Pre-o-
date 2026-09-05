
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Painel Mestre - Preço e Hora em Tempo Real</title>
    <style>
        body {
            background-color: #0b0e11;
            color: #eaecef;
            font-family: monospace;
            padding: 10px;
            margin: 0;
        }
        h2 { text-align: center; color: #f0b90b; font-size: 16px; margin-bottom: 10px; }
        
        .botoes-container {
            display: flex;
            flex-wrap: wrap;
            gap: 5px;
            justify-content: center;
            margin-bottom: 15px;
        }
        .btn-tempo {
            background-color: #1e2329;
            color: #eaecef;
            border: 1px solid #474d57;
            padding: 8px 10px;
            font-family: monospace;
            font-weight: bold;
            cursor: pointer;
            border-radius: 4px;
            font-size: 12px;
        }
        .btn-tempo.ativo {
            background-color: #f0b90b;
            color: #000;
            border-color: #f0b90b;
        }

        .alerta-box {
            background-color: #1e2329;
            border: 2px solid #f0b90b;
            padding: 10px;
            text-align: center;
            font-size: 14px;
            font-weight: bold;
            margin-bottom: 10px;
            border-radius: 5px;
        }
        .painel-info {
            background-color: #181a20;
            border: 1px solid #2b313a;
            padding: 12px;
            border-radius: 6px;
            font-size: 13px;
        }
        .linha-info {
            display: flex;
            justify-content: space-between;
            margin-bottom: 8px;
            border-bottom: 1px solid #2b313a;
            padding-bottom: 6px;
            align-items: center;
        }
        .medias-titulo {
            color: #f0b90b;
            margin-top: 10px;
            margin-bottom: 6px;
            font-weight: bold;
        }
        .medias-grid {
            color: #848e9c;
            font-size: 11px;
            line-height: 1.8;
        }
        .badge-compra {
            color: #0ecb81;
            font-weight: bold;
        }
        .badge-venda {
            color: #f6465d;
            font-weight: bold;
        }
    </style>
</head>
<body>

    <h2>BTUSD - CENTROS E HORAS (GRÁFICO NORMAL)</h2>

    <div class="botoes-container">
        <button class="btn-tempo ativo" onclick="mudarTempo('1m', this)">1m</button>
        <button class="btn-tempo" onclick="mudarTempo('2m', this)">2m</button>
        <button class="btn-tempo" onclick="mudarTempo('3m', this)">3m</button>
        <button class="btn-tempo" onclick="mudarTempo('4m', this)">4m</button>
        <button class="btn-tempo" onclick="mudarTempo('5m', this)">5m</button>
        <button class="btn-tempo" onclick="mudarTempo('6m', this)">6m</button>
        <button class="btn-tempo" onclick="mudarTempo('10m', this)">10m</button>
        <button class="btn-tempo" onclick="mudarTempo('12m', this)">12m</button>
        <button class="btn-tempo" onclick="mudarTempo('20m', this)">20m</button>
        <button class="btn-tempo" onclick="mudarTempo('30m', this)">30m</button>
    </div>

    <div id="status-sinal" class="alerta-box" style="color: #f0b90b;">
        🔍 CARREGANDO...
    </div>

    <div id="painel-detalhes" class="painel-info">
        Selecione um tempo acima...
    </div>

<script>
let tempoAtualBinance = '1m';
const periodosMa = [19, 38, 97, 191, 383, 575, 979];
let dadosGlobais = [];

function mudarTempo(intervalo, elemento) {
    tempoAtualBinance = intervalo;
    document.querySelectorAll('.btn-tempo').forEach(b => b.classList.remove('ativo'));
    elemento.classList.add('ativo');
    carregarDados();
}

// Mapeia os tempos para a API da Binance e define o fator de agrupamento se necessário
function obterConfiguracaoBinance(tempo) {
    switch(tempo) {
        let n = parseInt(tempo);
        // Se for nativo da Binance
        case '1m': return { baseApi: '1m', fator: 1 };
        case '3m': return { baseApi: '3m', fator: 1 };
        case '5m': return { baseApi: '5m', fator: 1 };
        case '10m': return { baseApi: '5m', fator: 2 };  // Duas velas de 5m = 10m
        case '20m': return { baseApi: '10m', fator: 2 }; // Duas velas de 10m = 20m
        case '30m': return { baseApi: '30m', fator: 1 };
        
        // Tempos customizados (agrupados do 1m)
        case '2m': return { baseApi: '1m', fator: 2 };
        case '4m': return { baseApi: '1m', fator: 4 };
        case '6m': return { baseApi: '1m', fator: 6 };
        case '12m': return { baseApi: '3m', fator: 4 };
        default: return { baseApi: '1m', fator: 1 };
    }
}

// Função para agrupar velas menores em tempos maiores caso a Binance não suporte nativamente
function agruparVelas(velasOriginal, fator) {
    if (fator <= 1) return velasOriginal;
    let velasAgrupadas = [];
    for (let i = 0; i < velasOriginal.length; i += fator) {
        let bloco = velasOriginal.slice(i, i + fator);
        if (bloco.length === 0) continue;
        
        let openTime = bloco[0][0];
        let open = bloco[0][1];
        let high = -Infinity;
        let low = Infinity;
        let close = bloco[bloco.length - 1][4];
        let volume = 0;
        
        bloco.forEach(v => {
            let h = parseFloat(v[2]);
            let l = parseFloat(v[3]);
            if (h > high) high = h;
            if (l < low) low = l;
            volume += parseFloat(v[5]);
        });
        
        velasAgrupadas.push([openTime, open, high.toString(), low.toString(), close, volume.toString()]);
    }
    return velasAgrupadas;
}

// Cálculo do centro no gráfico normal (Máxima + Mínima / 2)
function calcularCentro(velas) {
    if (!velas || velas.length === 0) return 0;
    let ultimaVela = velas[velas.length - 1];
    let maxima = parseFloat(ultimaVela[2]);
    let minima = parseFloat(ultimaVela[3]);
    return (maxima + minima) / 2;
}

function calcularMedia(velas, periodo) {
    if (velas.length < periodo) return null;
    let soma = 0;
    for (let i = velas.length - periodo; i < velas.length; i++) {
        soma += parseFloat(velas[i][4]);
    }
    return soma / periodo;
}

// Converte qualquer valor em dinheiro para o formato de relógio 24h (HH:MM:SS)
function valorParaHora(valor) {
    let numStr = valor.toFixed(2);
    let partes = numStr.split('.');
    let parteInteira = partes[0];
    let centavos = partes[1] || '00';

    if (parteInteira.length >= 4) {
        let horasBrutas = parseInt(parteInteira.substring(parteInteira.length - 4, parteInteira.length - 2));
        let minutos = parteInteira.substring(parteInteira.length - 2);
        
        let horasAjustadas = horasBrutas % 24;
        let hFormatado = String(horasAjustadas).padStart(2, '0');
        let mFormatado = parseInt(minutos) > 59 ? "59" : minutos;
        let sFormatado = parseInt(centavos) > 59 ? "59" : centavos;

        return `${hFormatado}:${mFormatado}:${sFormatado}`;
    }
    return "00:00:00";
}

async function carregarDados() {
    try {
        let config = obterConfiguracaoBinance(tempoAtualBinance);
        let res = await fetch(`https://api.binance.com/api/v3/klines?symbol=BTCUSDT&interval=${config.baseApi}&limit=1000`);
        let dadosBrutos = await res.json();
        
        // Aplica o agrupamento se for um tempo personalizado (ex: 2m, 4m, 6m, 12m)
        dadosGlobais = agruparVelas(dadosBrutos, config.fator);
        atualizarTela();
    } catch (e) {
        document.getElementById("status-sinal").innerText = "❌ ERRO AO CONECTAR NA BINANCE";
    }
}

function atualizarTela() {
    if (!dadosGlobais.length) return;

    let centro = calcularCentro(dadosGlobais);
    let precoAtual = parseFloat(dadosGlobais[dadosGlobais.length - 1][4]);

    let sinalTexto = "";
    let corSinal = "";

    if (precoAtual >= centro) {
        sinalTexto = `🚀 [${tempoAtualBinance.toUpperCase()}] PREÇO ACIMA DO CENTRO -> COMPRA FORTE!`;
        corSinal = "#0ecb81";
    } else {
        sinalTexto = `📉 [${tempoAtualBinance.toUpperCase()}] PREÇO ABAIXO DO CENTRO -> VENDA FORTE!`;
        corSinal = "#f6465d";
    }

    let caixaSinal = document.getElementById("status-sinal");
    caixaSinal.innerText = sinalTexto;
    caixaSinal.style.color = corSinal;
    caixaSinal.style.borderColor = corSinal;

    let precoHora = valorParaHora(precoAtual);
    let centroHora = valorParaHora(centro);

    let mediasHtml = "";
    periodosMa.forEach(p => {
        let maVal = calcularMedia(dadosGlobais, p);
        if (maVal) {
            let maHora = valorParaHora(maVal);
            let statusAlerta = (precoAtual >= maVal) ? 
                `<span class="badge-compra">🟢 [COMPRA]</span>` : 
                `<span class="badge-venda">🔴 [VENDA]</span>`;

            mediasHtml += `MA ${p}: <b>$ ${maVal.toFixed(2)}</b> &nbsp;|&nbsp; <span style="color: #f0b90b;">🕒 ${maHora}</span> &nbsp;→&nbsp; ${statusAlerta}<br>`;
        } else {
            mediasHtml += `MA ${p}: <i>Dados insuficientes</i><br>`;
        }
    });

    document.getElementById("painel-detalhes").innerHTML = `
        <div class="linha-info">
            <span>Preço do BTC:</span> <b>$ ${precoAtual.toFixed(2)} &nbsp;|&nbsp; <span style="color: #f0b90b;">🕒 ${precoHora}</span></b>
        </div>
        <div class="linha-info">
            <span>Centro da Vela (${tempoAtualBinance.toUpperCase()}):</span> <b>$ ${centro.toFixed(2)} &nbsp;|&nbsp; <span style="color: #0ecb81;">🕒 ${centroHora}</span></b>
        </div>
        <div class="linha-info">
            <span>Contagem Regressiva:</span> <b id="timer-txt" style="color: #f0b90b;">--:--</b>
        </div>
        <div class="medias-titulo">⏱️ MÉDIAS MESTRES (VALOR E HORA LADO A LADO):</div>
        <div class="medias-grid">${mediasHtml}</div>
    `;
    
    atualizarTimer();
}

function atualizarTimer() {
    const agora = new Date();
    let segundos = agora.getSeconds();
    let minutos = agora.getMinutes();
    
    let restoSegundos = 59 - segundos;
    let sFormatado = String(restoSegundos).padStart(2, '0');

    // Extrai o valor numérico do tempo (ex: '12m' vira 12)
    let minutosAlvo = parseInt(tempoAtualBinance);
    let restoMin = minutosAlvo - 1 - (minutos % minutosAlvo);
    if (restoMin < 0) restoMin = 0;
    let mFormatado = String(restoMin).padStart(2, '0');

    let elemTimer = document.getElementById("timer-txt");
    if (elemTimer) {
        elemTimer.innerText = `${mFormatado}:${sFormatado}`;
    }
}

carregarDados();
setInterval(carregarDados, 10000);
setInterval(atualizarTimer, 1000);
</script>

</body>
</html>
