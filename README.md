# Actograma Studio

Análise e interpretação automática de actigrafia em uma única página HTML, sem instalação, sem servidor e sem envio de dados.

Você abre o arquivo `.txt` exportado do ActStudio (Condor Instruments ActTrust) e recebe o actograma em dupla plotagem, os parâmetros de sono noite a noite — na nomenclatura do **Consenso Brasileiro de Actigrafia** (FPR, FPS, TTS), com a **decisão graduada de HIS/HFS** por atividade + evento + luz + temperatura —, o dia médio de 24 h, a análise circadiana paramétrica (cosinor) e não-paramétrica (IS, IV, RA, CFI, L5, M10), periodograma χ², deriva de fase, comparação dias úteis × livres com jetlag social, uma **interpretação automatizada** que classifica cada parâmetro contra referências publicadas e uma **triagem de transtornos do sono** que confronta o padrão actigráfico com os critérios da CIDS-3-TR. Os resultados podem ser exportados em **PDF**, **JSON** e **CSV**.

> **Privacidade.** Todo o processamento acontece no navegador, em JavaScript. Nenhum byte do registro sai da máquina — não há back-end, upload, telemetria ou dependência externa. O `index.html` é autocontido e funciona offline, inclusive com duplo clique a partir do disco.

---

## Como usar

1. Baixe o `index.html` (ou clone o repositório) e abra-o em um navegador moderno — Chrome, Edge, Firefox ou Safari.
2. Arraste o arquivo `.txt` exportado do ActStudio para a área indicada, ou clique em **Abrir arquivo…**.
3. Para conhecer a ferramenta sem dados reais, clique em **Ver com dados de demonstração**: um registro sintético de 8 dias é gerado no próprio navegador.

Também é possível publicar o arquivo em qualquer hospedagem estática (GitHub Pages, por exemplo) — continua sendo 100% client-side.

### Formato de entrada

Exportação padrão do ActStudio (Condor Instruments), com cabeçalho `Condor Instruments Report` e colunas separadas por `;`:

```
DATE/TIME;MS;EVENT;TEMPERATURE;EXT TEMPERATURE;ORIENTATION;PIM;PIMn;TAT;TATn;ZCM;ZCMn;LIGHT;AMB LIGHT;...;STATE
02/03/2026 00:00:00;0;0;34.17;27.00;0;461;0;69;0;0;0;0.20;0.00;...;0
```

- Colunas usadas: `DATE/TIME`, `PIM`, `TAT`, `ZCM`, `TEMPERATURE`, `LIGHT`, `EVENT`. As demais são ignoradas.
- A época é lida de `INTERVAL` no cabeçalho ou inferida da mediana dos intervalos. Registros com época < 60 s são reamostrados para 1 min (soma dos canais de atividade, média dos demais).
- Os horários são tratados como relógio de parede, sem conversão de fuso.
- Lacunas de registro são preenchidas com épocas ausentes e mascaradas.

### Parâmetros ajustáveis

Alterar qualquer opção recalcula tudo instantaneamente:

| Parâmetro | Opções | Padrão |
|---|---|---|
| Algoritmo de escoragem | Cole-Kripke, Sadeh | Cole-Kripke |
| Canal exibido no actograma | PIM, ZCM, TAT | PIM |
| Máscara de off-wrist por temperatura | ligada/desligada, limiar de 20–30 °C | ligada, 25 °C |
| Luz no actograma | ligada/desligada | ligada |

---

## O que é calculado

### Parâmetros de sono (por episódio e média do período)

A nomenclatura segue o Consenso Brasileiro de Actigrafia (ABS, 2021):

| Parâmetro | Definição adotada |
|---|---|
| **Deitar** ("luzes apagadas") | Marcador de evento do actígrafo, quando existe um até 180 min antes do início do sono; caso contrário, estimado por atividade (ver abaixo) |
| **HIS (início do sono)** | Primeira janela de 10 min com no máximo 1 época de vigília |
| **HFS (fim do sono)** | Fim do episódio principal detectado |
| **Latência de sono (SOL)** | HIS − deitar |
| **FPR** (fase principal de repouso) | Deitar → fim do episódio — substitui o impreciso "tempo na cama" |
| **FPS** (fase principal de sono) | HIS → HFS |
| **TTS** (tempo total de sono) | FPS − WASO (épocas de sono entre HIS e HFS) |
| **WASO** | Vigília entre HIS e HFS |
| **Eficiência de sono (ES)** | TTS / FPR × 100, excluídas épocas mascaradas |
| **Número de despertares** | Sequências de vigília ≥ 1 min após o HIS (também reportadas as ≥ 5 min) |
| **Índice de fragmentação** | Transições sono↔vigília por hora de TTS |

**Decisão graduada de HIS e HFS (Quadros 1 e 2 do Consenso).** A atividade é a variável-base; cada limite é validado pelos sinais auxiliares disponíveis no próprio registro — botão de evento, queda/subida da luz (janelas de 45 min, razão 3×) e temperatura do punho (subida ≥ 0,3 °C ao adormecer, queda ao despertar). Três sinais além da atividade = decisão **muito forte**; dois = **forte**; um ou nenhum = **moderada**. O nível aparece por noite na tabela (badges MF/F/M) e em todas as exportações.

Os episódios são detectados automaticamente: densidade de sono (média móvel de ±15 min) > 0,5; blocos separados por < 30 min são fundidos, e também os separados por até 180 min quando a atividade no intervalo permanece abaixo do limiar de repouso — assim um despertar longo dentro da cama não parte a noite em dois. Blocos < 10 min são descartados (episódios secundários contam a partir de 10 min, conforme o Consenso). Em cada janela meio-dia → meio-dia, o maior bloco com ≥ 2 h é o **episódio principal (noite)**; blocos de 10–240 min são registrados como **cochilos**, exceto os cortados pelas bordas do registro.

Na ausência de marcador de evento, o deitar é estimado recuando até 120 min a partir do bloco de repouso enquanto a atividade suavizada (média móvel de ±5 min de PIM) permanece abaixo do **limiar de repouso** — 20% da atividade média da janela M10 do próprio indivíduo, o que dispensa calibração para a escala de contagens do aparelho. As exportações trazem a coluna `origem_deitar` (`evento` ou `atividade`); a estimativa por atividade é um *proxy* e tende a subestimar a latência em relação ao diário de sono.

Episódios cortados pelo início ou pelo fim do registro são marcados como **incompletos**, exibidos na tabela e excluídos de todas as médias, já que sua duração real é desconhecida.

Além do índice de fragmentação por transições sono↔vigília (sem valor de referência publicado), é calculado o **índice de fragmentação no padrão Actiware** — % de épocas móveis + % de bouts imóveis de 1 min, com época móvel definida por contagens ≥ (segundos da época)/15 sobre ZCM —, que é a métrica à qual se aplica o limiar de 35 da literatura, e a **atividade motora média** (ZCM/min entre o início e o fim do sono).

Horários de deitar, início, fim e meio do sono são resumidos por **média circular**, com o desvio-padrão circular do meio do sono como medida de regularidade.

As noites completas são separadas em **dias úteis × dias livres** pelo dia em que o sono termina (sábado e domingo contam como livres — a divisão presume fim de semana livre e não vale para escalas atípicas). Disso saem o **jetlag social** (diferença do meio do sono livre − útil), a **extensão do TTS nos dias livres** e a detecção de **despertar induzido** (fim do sono rígido nos dias úteis, DP < 45 min, com extensão nos livres). A **deriva de fase** é a inclinação da regressão linear dos meios do sono desdobrados (min/dia), e o **periodograma χ² de Sokolove–Bushell** (PIM em caixas de 5 min, varredura 20–28 h) estima o período dominante do ritmo — em ritmo sincronizado, 24 h.

### Ritmo circadiano

| Métrica | Descrição |
|---|---|
| **Dia médio** | Perfil de 24 h por minuto do dia (0–1439): atividade média (PIM), probabilidade média de sono e luz média, com épocas mascaradas excluídas |
| **L5** | Média das 5 h consecutivas de menor atividade no dia médio, com horário de início |
| **M10** | Média das 10 h consecutivas de maior atividade no dia médio, com horário de início |
| **RA** | Amplitude relativa: (M10 − L5) / (M10 + L5) |
| **IS** | Estabilidade interdiária, sobre médias horárias de PIM |
| **IV** | Variabilidade intradiária, sobre médias horárias de PIM |
| **CFI** | Índice de função circadiana: média de IS, do IV normalizado [(2 − IV)/2] e de RA, cada termo limitado a 0–1 |
| **Cosinor** | MESOR, amplitude, acrofase e R² do ajuste de 24 h sobre log₁₀(PIM+1) |

L5 e M10 usam janelas móveis circulares sobre o dia médio por minuto. IS, IV e, por consequência, o CFI exigem ≥ 48 h válidas; abaixo de 3 dias de registro a interface sinaliza baixa confiabilidade (a AASM recomenda 72 h a 14 dias).

O **CFI** varia de 0 (ausência de ritmo circadiano detectável) a 1 (ritmo robusto e estável) e só é reportado quando IS, IV e RA estão todos disponíveis.

### Controle de qualidade

Épocas com temperatura cutânea abaixo do limiar (off-wrist) ou correspondentes a lacunas do registro são mascaradas, destacadas em vermelho no actograma e excluídas de todos os cálculos.

---

## Interpretação automatizada

> **Não existem valores de normalidade universais em actigrafia.** Os limiares dependem do aparelho, do modo de aquisição (PIM/TAT/ZCM) e do algoritmo de escoragem, e os que este programa usa **não foram derivados com o ActTrust**. A interpretação é uma leitura orientadora que sempre acompanha a referência que aplicou e as ressalvas do caso — nunca um laudo.

Cada parâmetro recebe uma classificação (**normal**, **limítrofe**, **alterado**, **descritivo** quando não há limiar publicado, ou **sem dados**), junto da referência adotada. Métricas cujo limiar existe mas não foi validado para este aparelho são marcadas como **orientador**.

### Limiares de sono

Critérios actigráficos quantitativos de Natale et al. (Motionlogger, 2009; Actiwatch, 2014) e limiares clínicos difundidos:

| Parâmetro | Normal | Limítrofe | Alterado | Referência |
|---|---|---|---|---|
| TST | 7–9 h | 6–7 h ou > 9 h | < 6 h | 7–9 h em adultos; QAC considera anormal ≤ 440 min |
| SOL | < 15 min | 15–29 min | ≥ 30 min | < 14 min (Actiwatch), < 12 min (Motionlogger); ≥ 30 min = insônia de início |
| ES | ≥ 85% | 80–85% | < 80% | > 87% (Actiwatch), > 92% (Motionlogger) |
| WASO | < 30 min | 30–40 min | > 40 min | < 40 min (Actiwatch), < 25 min (Motionlogger) |
| Despertares ≥ 5 min | < 2 | — | ≥ 2 | ≥ 2 episódios é critério de anormalidade |
| Índice de fragmentação | < 35 | — | ≥ 35 | Actiwatch |
| Atividade motora média | < 16 ZCM/min | — | ≥ 16 | Motionlogger |

### Faixas circadianas

Sem cutoffs clínicos validados — são faixas típicas de coortes de adultos saudáveis, e IS e RA diminuem enquanto IV aumenta com a idade:

| Parâmetro | Normal | Limítrofe | Alterado |
|---|---|---|---|
| IS | ≥ 0,6 | 0,4–0,6 | < 0,4 |
| IV | < 1,0 | 1,0–1,5 | > 1,5 |
| RA | ≥ 0,85 | 0,80–0,85 | < 0,80 |
| Fase (meio de L5) | 01:00–05:00 | 05:00–08:00 (atraso) ou 21:00–01:00 (avanço) | horário diurno |
| Regularidade (DP do meio do sono) | < 1 h | 1–1,5 h | > 1,5 h |

CFI e os valores absolutos de L5 e M10 são apresentados como **descritivos**: o CFI não tem cutoff validado, e as contagens não são comparáveis entre aparelhos ou modos de aquisição — o que se interpreta é a direção e a evolução intraindividual. A acrofase é avaliada pela coerência com a janela M10 e pela robustez do ajuste (R²).

### Triagem de transtornos do sono (CIDS-3-TR)

Regras fixas confrontam o padrão actigráfico com a parte dos critérios diagnósticos que a actigrafia consegue examinar, e cada hipótese sai como **compatível**, **possível**, **sem sinais actigráficos** ou **não avaliável**, sempre listando os critérios verificados (✓/✗) e os que **dependem da avaliação clínica** — queixa, duração ≥ 3 meses e prejuízo funcional nunca são verificáveis pelo actígrafo, e a CIDS-3-TR exige diário de sono acompanhando o registro.

| Hipótese | Sinais actigráficos exigidos |
|---|---|
| **Atraso de fase (DSWPD)** | Início do sono ≥ 01:00, fase estável (DP ≤ 2 h, sem deriva), noites não partidas, sono adequado quando livre |
| **Avanço de fase (ASWPD)** | Início ≤ 21:00 e fim ≤ 05:00, fase estável |
| **Ritmo irregular (ISWRD)** | ≥ 3 episódios de sono/24 h ou ausência de episódio principal, com IS < 0,5 ou DP do meio do sono > 2 h |
| **Ritmo ≠ 24 h (N24SWD)** | Deriva progressiva ≥ 15 min/dia (r² ≥ 0,6) + período ≠ 24 h no periodograma; exige ≥ 14 dias |
| **Insônia crônica (padrão)** | SOL ≥ 30 min ou má manutenção (WASO > 40 min / ≥ 2 despertares ≥ 5 min com ES < 85%, ou noites partidas) em ≥ 40% das noites ≈ ≥ 3×/semana, com oportunidade adequada |
| **Sono insuficiente (SSI)** | TTS útil < 7 h + extensão ≥ 1 h nos dias livres + despertar induzido ou sono eficiente |
| **Sono fragmentado (inespecífico)** | WASO > 40 min + ≥ 2 despertares ≥ 5 min + fragmentação do ritmo — investigar AOS/MPM/dor |

A AASM classifica o apoio da actigrafia nesses cenários como recomendação **condicional** ("sugerimos") e recomenda **fortemente não** usá-la no lugar da EMG para movimentos periódicos dos membros — a triagem nunca propõe esse diagnóstico.

### Achados e qualidade do registro

Os itens classificados são agregados em padrões nomeados — latência prolongada, sono fragmentado, eficiência reduzida, sono curto, ritmo de repouso-atividade irregular, contraste dia-noite reduzido, desvio de fase, horários irregulares e cochilos frequentes — e sintetizados em um parágrafo.

Em paralelo, o programa sinaliza o que compromete a leitura: registro abaixo dos 5–7 dias consensuais (14 dias para a latência), poucas noites completas, proporção elevada de épocas mascaradas, ausência de marcador de evento (que faz a latência ser subestimada), episódios incompletos excluídos das médias e noites partidas em episódios separados — quando o episódio principal contém menos de 70% do sono da janela de 24 h, TIB, TST e eficiência se referem apenas a ele.

---

## Exportação dos resultados

Botão **Exportar** na barra superior:

| Formato | Conteúdo |
|---|---|
| **Relatório PDF** | Relatório paginado em A4: resumo do sono e do ritmo circadiano, interpretação automatizada com achados e ressalvas, triagem de transtornos com critérios verificados, dados do dispositivo, actograma, dia médio, tabela por episódio com decisão graduada e notas metodológicas |
| **JSON** | Objeto único com configuração, registro, todos os episódios, médias, horários, métricas circadianas, a interpretação completa e as três séries do dia médio (1440 pontos cada) |
| **CSV — noites** | Uma linha por episódio (noites e cochilos), linha de média, bloco de resumo com IS, IV, RA, CFI, L5, M10 e cosinor, e blocos com a interpretação, os padrões e as ressalvas |
| **CSV — dia médio** | Perfil de 24 h minuto a minuto: atividade, probabilidade de sono e luz |
| **CSV — épocas** | Registro completo época a época, com escore de sono, máscara e marcação de repouso |

Cada gráfico também pode ser exportado isoladamente em **PNG** (botão *Exportar PNG* no cabeçalho do respectivo cartão) e a página é imprimível diretamente pelo navegador.

O PDF é gerado inteiramente no navegador, sem biblioteca externa: um gerador próprio monta objetos PDF 1.4 com fontes Helvetica (WinAnsi) e embute os gráficos como imagens JPEG (DCTDecode), sempre renderizados em paleta clara sobre fundo branco. Os CSV usam `;` como separador e vírgula decimal, prontos para o Excel em português.

---

## Métodos e referências

- **Escoragem sono-vigília** — aplicada ao canal ZCM em épocas de 1 min.
  - Cole RJ, Kripke DF, Gruen W, Mullaney DJ, Gillin JC. Automatic sleep/wake identification from wrist activity. *Sleep*. 1992;15(5):461-9.
  - Sadeh A, Sharkey KM, Carskadon MA. Activity-based sleep-wake identification. *Sleep*. 1994;17(3):201-7.
- **Análise circadiana não-paramétrica (IS, IV, L5, M10, RA)** — Van Someren EJW, Swaab DF, Colenda CC, Cohen W, McCall WV, Rosenquist PB. Bright light therapy: improved sensitivity to its effects on rest-activity rhythms in Alzheimer patients by application of nonparametric methods. *Chronobiol Int*. 1999;16(4):505-18.
- **Índice de função circadiana (CFI)** — Ortiz-Tudela E, Martinez-Nicolas A, Campos M, Rol MÁ, Madrid JA. A new integrated variable based on thermometry, actimetry and body position (TAP) to evaluate circadian system status in humans. *PLoS Comput Biol*. 2010;6(11):e1000996.
- **Cosinor** — ajuste por mínimos quadrados de y = MESOR + A·cos(2π·t/24 h − φ).
- **Critérios quantitativos actigráficos usados na interpretação** — Natale V, Plazzi G, Martoni M. Actigraphy in the assessment of insomnia: a quantitative approach. *Sleep*. 2009;32(6):767-71; e Natale V, Léger D, Martoni M, Bayon V, Erbacci A. The role of actigraphy in the assessment of primary insomnia: a retrospective study. *Sleep Med*. 2014;15(1):111-5. Derivados com Motionlogger e Actiwatch — **não com o ActTrust**.
- **Padronização brasileira** — Consenso Brasileiro de Actigrafia, Associação Brasileira do Sono, 2021: base da nomenclatura adotada (FPR no lugar de "tempo na cama", FPS, TTS = FPS − WASO, ES = TTS/FPR × 100, cochilos ≥ 10 min), da decisão graduada de HIS/HFS com evento + atividade + luz + temperatura e do mínimo de 5–7 dias de registro, sem fixar valores de normalidade.
- **Critérios diagnósticos usados na triagem** — Classificação Internacional dos Transtornos do Sono, 3ª ed. revisada (ICSD-3-TR, AASM 2023), e Smith MT et al. Use of actigraphy for the evaluation of sleep disorders and circadian rhythm sleep-wake disorders: an AASM clinical practice guideline. *J Clin Sleep Med*. 2018;14(7):1231-7 (recomendações condicionais para insônia, transtornos circadianos, TTS pré-TLMS e sono insuficiente; recomendação forte contra o uso no lugar da EMG para MPM).
- **Periodograma** — Sokolove PG, Bushell WN. The chi square periodogram: its utility for analysis of circadian rhythms. *J Theor Biol*. 1978;72(1):131-60.
- **Uso clínico da actigrafia** — Smith MT, McCrae CS, Cheung J, et al. Use of actigraphy for the evaluation of sleep disorders and circadian rhythm disorders: an AASM clinical practice guideline. *J Clin Sleep Med*. 2018;14(7):1231-7.

Cole-Kripke e Sadeh foram derivados contra polissonografia em actígrafos AMI; a aplicação a outros dispositivos assume comparabilidade das contagens ZCM.

---

## Estrutura do projeto

```
index.html    aplicação inteira: CSS, motor de análise (CORE),
              interpretação (INTERP) e interface/exportações
README.md
```

Os blocos `CORE` (parsing, grade temporal, escoragem, detecção de episódios, cosinor, NPCRA, dia médio) e `INTERP` (classificação, padrões, síntese e ressalvas) são independentes do DOM e podem ser reaproveitados em Node.js ou em testes.

---

## Aviso

Ferramenta de apoio à pesquisa e à leitura clínica. **Não substitui polissonografia nem julgamento médico.** A actigrafia infere sono a partir da imobilidade e tende a superestimar o tempo total de sono e a eficiência e a subestimar o WASO e a latência. Os limiares e regras de detecção descritos acima são escolhas explícitas da implementação e podem divergir de outros softwares de actigrafia — verifique-os antes de usar os resultados em pesquisa ou assistência.

A interpretação automatizada não é um laudo: é uma leitura estruturada dos números contra referências que **não foram validadas para este aparelho**, sem acesso ao diário de sono, à queixa, à idade, aos medicamentos ou ao contexto do paciente. A comparação intraindividual — pré e pós-tratamento, dias úteis versus dias livres — é o uso mais robusto da actigrafia, e a leitura do actograma continua sendo indispensável.
