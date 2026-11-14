# Web Scrapping Eleitoral

Ferramenta em Python para coletar e analisar a cobertura jornalística sobre candidatos à Presidência da República nas eleições de 2026. São dois scripts que rodam em sequência: o primeiro localiza notícias sobre um candidato em 28 veículos brasileiros; o segundo baixa o texto integral dessas notícias e mede quais palavras mais aparecem.

**Autor:** Guilherme Berlanga
**Período dos testes:** 12 a 28 de setembro de 2026
**Ambiente dos testes:** Linux, Python 3.14

---

## 1. O que foi o projeto

O objetivo era responder a uma pergunta simples: **o que a imprensa está dizendo sobre cada candidato, e isso conversa com o que o candidato propõe?**

Ler centenas de matérias à mão não é viável, então o trabalho foi dividido em três partes:

1. **Coleta** — um script busca e cataloga as notícias sobre o candidato, veículo por veículo.
2. **Extração e contagem** — outro script abre cada link, extrai o texto completo da matéria e conta a frequência das palavras.
3. **Leitura comparada com IA** — o material coletado é entregue a um assistente de IA junto com o plano de governo do candidato.

### Como o material foi usado (caso Flávio Bolsonaro)

O conteúdo coletado sobre Flávio Bolsonaro (388 notícias, 376 com texto extraído) foi entregue a uma IA. Pedi que ela analisasse esse material **em conjunto com o plano de governo do candidato** e produzisse um **sumário do plano**. A finalidade do sumário era servir de base para uma análise coerente com o que estava sendo dito na mídia: ter, lado a lado, o que o plano propõe e os temas que a cobertura jornalística de fato destacava.

Essa etapa foi feita em conversa com a IA, fora dos scripts. O sumário do plano de governo não faz parte deste pacote; aqui estão a ferramenta e os dados de mídia que a alimentaram.

A mesma coleta foi repetida para outros três candidatos (Lula, Renan Santos e Ronaldo Caiado), o que permite aplicar o mesmo método a qualquer um deles.

---

## 2. O que a ferramenta faz

### Script 1 — `busca_noticias_candidato.py`

Pergunta o nome do candidato e:

- pesquisa no DuckDuckGo, restringindo a busca ao domínio de cada um dos 28 veículos (G1, O Globo, UOL, Folha, Estadão, CNN Brasil, Poder360, Metrópoles, Veja, CartaCapital, Gazeta do Povo, Jovem Pan, Agência Brasil, entre outros);
- faz duas buscas por veículo, uma geral e outra exigindo o nome no título, e ordena os resultados por relevância (menção no título vale mais que no corpo);
- descarta o que não é jornalismo, mesmo que apareça na busca (Wikipédia, redes sociais, YouTube, agregadores);
- testa cada link e remove os quebrados. Respostas HTTP 403 e 429 são tratadas como provável bloqueio anti-robô, e não como link morto;
- opcionalmente, lê planilhas de execuções anteriores do mesmo candidato para não catalogar a mesma notícia duas vezes;
- grava uma planilha `noticias_<Candidato>_<data>_<hora>.xlsx` com as notícias agrupadas por veículo.

### Script 2 — `analisar_conteudo_noticias.py`

Recebe uma ou mais planilhas do Script 1 e:

- combina os links e elimina duplicados entre planilhas;
- baixa cada página e extrai o texto da matéria por dois métodos (trafilatura e BeautifulSoup), ficando com o resultado mais completo;
- visita a página inicial do site antes do artigo, o que reduz bloqueios, e faz uma segunda rodada só com os links que falharam;
- conta a frequência das palavras, ignorando palavras de ligação do português e termos com menos de três letras;
- grava uma planilha `analise_noticias_<Candidato>_<data>_<hora>.xlsx` com três abas (ver seção 4);
- se a planilha não puder ser gerada, salva os dados coletados em um arquivo JSON de backup, para que a coleta não precise ser refeita.

---

## 3. Como executar

### Requisitos

- Python 3.10 ou superior
- Conexão com a internet

### Instalação

```bash
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### Passo 1 — buscar as notícias

```bash
python3 scripts/busca_noticias_candidato.py
```

O script pergunta, nesta ordem:

1. o nome do candidato;
2. se deve consultar uma pasta com planilhas antigas para não repetir notícias;
3. a pasta onde salvar o resultado.

### Passo 2 — analisar o conteúdo

```bash
python3 scripts/analisar_conteudo_noticias.py
```

O script pergunta:

1. o caminho de cada planilha gerada no Passo 1 (Enter em branco encerra a lista);
2. o nome do candidato, usado no título e no nome do arquivo;
3. a pasta onde salvar o resultado.

A coleta dos textos é a etapa mais demorada: leva alguns minutos para cerca de 200 notícias.

---

## 4. Como analisar os resultados

### Planilha de busca (`noticias_*.xlsx`)

Uma aba, `Noticias`, com três colunas: **Jornal**, **Título da Notícia** e **Link**. Cada veículo tem uma linha de cabeçalho com a quantidade de notícias válidas, e as notícias ficam agrupadas abaixo dela (o grupo pode ser recolhido no Excel). Serve para ver quais veículos mais cobriram o candidato e para conferir as fontes uma a uma.

### Planilha de análise (`analise_noticias_*.xlsx`)

| Aba | Conteúdo | Para que serve |
|---|---|---|
| `Noticias_Texto` | Jornal, título, link, status e texto completo (ou motivo da falha) | Ler as matérias e entregar o texto a uma IA para resumo ou comparação |
| `Frequencia_Completa` | Todas as palavras e quantas vezes aparecem | Procurar temas específicos (ex.: "segurança", "economia") |
| `Top20` | As 20 palavras mais citadas e um gráfico de barras | Visão rápida dos assuntos dominantes |

### Roteiro de leitura sugerido

1. **Comece pelo `Top20`.** Além do próprio nome do candidato, veja quais outros nomes e temas aparecem. Adversários e assuntos específicos no topo mostram em que contexto a imprensa fala dele.
2. **Aprofunde na `Frequencia_Completa`.** Busque as palavras-chave do plano de governo e compare: o tema central do plano aparece muito ou pouco na cobertura?
3. **Volte ao texto.** Uma palavra frequente não diz se a menção é favorável ou crítica. Filtre a aba `Noticias_Texto` e leia as matérias em que o termo aparece.
4. **Confira a coluna Status.** As linhas com "Falha" ficaram fora da contagem.
5. **Cruze com o plano de governo.** Entregue a aba `Noticias_Texto` e o plano a uma IA e peça o sumário comparado, como descrito na seção 1.

---

## 5. Testes realizados

### Busca

| Candidato | Data | Links válidos | Veículos com resultado (de 28) |
|---|---|---|---|
| Flávio Bolsonaro | 12/09/2026 | 196 | 22 |
| Flávio Bolsonaro | 17/09/2026 (21h03) | 147 | 23 |
| Flávio Bolsonaro | 17/09/2026 (21h15) | 45 | 10 |
| Renan Santos | 21/09/2026 | 206 | 19 |
| Lula | 22/09/2026 | 199 | 22 |
| Ronaldo Caiado | 28/09/2026 | 235 | 23 |

As três execuções de Flávio Bolsonaro usaram o recurso de histórico: cada nova rodada trouxe apenas notícias ainda não catalogadas. Somadas, dão as 388 notícias analisadas.

### Análise de conteúdo

| Candidato | Notícias | Texto extraído | Falhas | Palavras distintas |
|---|---|---|---|---|
| Flávio Bolsonaro | 388 | 376 (96,9%) | 12 | 13.935 |
| Renan Santos | 206 | 202 (98,1%) | 4 | 11.197 |
| Lula | 199 | 191 (96,0%) | 8 | 12.634 |
| Ronaldo Caiado | 235 | 233 (99,1%) | 2 | 9.651 |

As 26 falhas tiveram três causas: resposta em formato compactado que o script não consegue ler (12), bloqueio do site com HTTP 403 (11) e página sem texto extraível, por paywall ou bloqueio (3).

### Dez palavras mais citadas em cada teste

- **Flávio Bolsonaro:** flávio, bolsonaro, presidente, lula, senador, candidato, brasil, jair, governo, federal
- **Lula:** lula, presidente, brasil, governo, bolsonaro, são, flávio, país, contra, federal
- **Renan Santos:** renan, santos, candidato, missão, eleitoral, brasil, lula, campanha, são, afirmou
- **Ronaldo Caiado:** caiado, candidato, ronaldo, lula, psd, governador, brasil, goiás, flávio, presidente

---

## 6. Limitações

- **Frequência não é opinião.** A contagem mostra do que se fala, não se o tom é positivo ou negativo.
- **A busca depende do DuckDuckGo.** O teto é de 500 resultados por veículo, mas o buscador costuma devolver bem menos. A amostra não representa toda a cobertura.
- **Alguns veículos bloqueiam a coleta.** Sites com paywall ou proteção anti-robô podem ficar sub-representados.
- **Páginas-índice entram na amostra.** Páginas de perfil do candidato em um portal listam manchetes variadas, inclusive sobre outros candidatos, e isso entra na contagem.
- **Palavras de pouco significado ainda aparecem.** Termos como "são", "feira" (de "segunda-feira") e "você" passam pelo filtro e devem ser desconsiderados na leitura.
- **Retrato de um momento.** Cada planilha reflete o que estava publicado na data da coleta.

---

## 7. Estrutura do pacote

```
Web_Scrapping_Eleitoral/
├── README.md
├── requirements.txt
├── scripts/
│   ├── busca_noticias_candidato.py
│   └── analisar_conteudo_noticias.py
└── resultados_testes/
    ├── 1_busca/        6 planilhas de links (saída do Script 1)
    └── 2_analise/      4 planilhas de texto e frequência (saída do Script 2)
```

---

## 8. Uso responsável

A ferramenta acessa apenas páginas públicas, com pausas entre as requisições. Os textos das matérias pertencem aos respectivos veículos e foram coletados para estudo e análise; não devem ser republicados. O projeto não tem vínculo com partidos, candidatos ou campanhas.
