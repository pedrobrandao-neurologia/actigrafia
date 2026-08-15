# Actograma Studio

Análise e interpretação automática de actigrafia em uma única página HTML, sem instalação, sem servidor e sem envio de dados.

Você abre o arquivo `.txt` exportado do ActStudio (Condor Instruments ActTrust) e recebe o actograma em dupla plotagem, os parâmetros de sono noite a noite, o dia médio de 24 h e a análise circadiana paramétrica (cosinor) e não-paramétrica (IS, IV, RA, CFI, L5, M10). Os resultados podem ser exportados em **PDF**, **JSON** e **CSV**.

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

| Parâmetro | Definição adotada |
|---|---|
| **Deitar** ("luzes apagadas") | Marcador de evento do actígrafo, quando existe um até 180 min antes do início do sono; caso contrário, estimado por atividade (ver abaixo) |
| **Início do sono** | Primeira janela de 10 min com no máximo 1 época de vigília |
| **Latência de sono (SOL)** | Início do sono − deitar |
| **TIB** | Deitar → fim do episódio |
| **TST** | Épocas escoradas como sono dentro do TIB |
| **WASO** | Vigília entre o início do sono e o fim do episódio |
| **Eficiência de sono (ES)** | TST / TIB, excluídas épocas mascaradas |
| **Número de despertares** | Sequências de vigília ≥ 1 min após o início do sono (também reportadas as ≥ 5 min) |
| **Índice de fragmentação** | Transições sono↔vigília por hora de TST |

Os episódios são detectados automaticamente: densidade de sono (média móvel de ±15 min) > 0,5, fusão de intervalos separados por < 60 min e descarte de blocos < 15 min. Em cada janela meio-dia → meio-dia, o maior bloco com ≥ 2 h é o **episódio principal (noite)**; blocos de 15–240 min são registrados como **cochilos**.

Na ausência de marcador de evento, o deitar é estimado recuando até 90 min a partir do bloco de repouso enquanto a densidade local de sono permanece ≥ 0,2 — o período de acomodação, em que já há imobilidade, mas ainda intermitente. O recuo não atravessa vigília contínua > 30 min nem o episódio anterior. As exportações trazem a coluna `origem_deitar` (`evento` ou `atividade`); a estimativa por atividade é um *proxy* e tende a subestimar a latência em relação ao diário de sono.

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

## Exportação dos resultados

Botão **Exportar** na barra superior:

| Formato | Conteúdo |
|---|---|
| **Relatório PDF** | Laudo paginado em A4: resumo do sono e do ritmo circadiano, dados do dispositivo, actograma, dia médio, tabela por episódio e notas metodológicas |
| **JSON** | Objeto único com configuração, registro, todos os episódios, médias, métricas circadianas e as três séries do dia médio (1440 pontos cada) |
| **CSV — noites** | Uma linha por episódio (noites e cochilos), linha de média e bloco de resumo com IS, IV, RA, CFI, L5, M10 e cosinor |
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
- **Uso clínico da actigrafia** — Smith MT, McCrae CS, Cheung J, et al. Use of actigraphy for the evaluation of sleep disorders and circadian rhythm disorders: an AASM clinical practice guideline. *J Clin Sleep Med*. 2018;14(7):1231-7.

Cole-Kripke e Sadeh foram derivados contra polissonografia em actígrafos AMI; a aplicação a outros dispositivos assume comparabilidade das contagens ZCM.

---

## Estrutura do projeto

```
index.html    aplicação inteira: CSS, motor de análise (CORE) e interface/exportações
README.md
```

O bloco `CORE` é independente do DOM (parsing, grade temporal, escoragem, detecção de episódios, cosinor, NPCRA, dia médio) e pode ser reaproveitado em Node.js ou em testes.

---

## Aviso

Ferramenta de apoio à pesquisa e à leitura clínica. **Não substitui polissonografia nem julgamento médico.** A actigrafia infere sono a partir da imobilidade e tende a superestimar o tempo total de sono em sono fragmentado. Os limiares e regras de detecção descritos acima são escolhas explícitas da implementação e podem divergir de outros softwares de actigrafia — verifique-os antes de usar os resultados em pesquisa ou assistência.
