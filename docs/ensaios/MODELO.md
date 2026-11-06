# Ensaio H00 · Nome curto do ensaio · DD/MM/AAAA

> Copie este arquivo como `AAAA-MM-DD-hNN-nome.md` (ex.: `2026-10-30-h02-desempenho-visao.md`),
> preencha e apague as linhas de instrução (as que começam com `>`).
> O ID `H..` e os critérios vêm do [plano de testes do hardware](../PLANO_TESTES_HW.md).

- **Responsáveis:** quem executou · quem conferiu
- **Tarefa do cronograma:** T..
- **Requisitos:** RF.. / RNF..
- **Resultado:** ✔ aprovado · ✘ reprovado · ◐ parcial

## Objetivo

> Uma ou duas frases: o que este ensaio prova.

## Montagem

| Item | Qual |
|---|---|
| Placas e módulos | ex.: Raspberry Pi 4B 4 GB, webcam USB (a OV5647 ainda não chegou) |
| Alimentação | ex.: fonte oficial 5 V 3 A pelo USB-C |
| Instrumentos | ex.: multímetro, `vcgencmd` |
| Software | repositório e commit (`git log -1 --format=%h`), versão do firmware (do STATUS) |
| Ambiente | ex.: bancada, sala a ~25 °C, luz ambiente |

> Foto da montagem: `AAAA-MM-DD-hNN-montagem.jpg`, na mesma pasta.

## Procedimento

> Os passos e comandos que foram realmente executados. Se algo saiu diferente do plano,
> escreva aqui o quê e por quê (desvio).

1. …
2. …

## Critérios

| Critério | Esperado | Visto | |
|---|---|---|---|
| ex.: FPS médio | ≥ 10 | 14,2 | ✔ |
| ex.: Temperatura máxima | < 80 °C | 71,5 °C | ✔ |
| … | | | |

## Dados

- **Arquivos:** logs e medições, em `AAAA-MM-DD-hNN-*.log` ou `.csv`.
  Gravações grandes (vídeo, dezenas de MB) ficam fora do git; anote aqui onde estão.
- **Observações:** o que chamou a atenção, mesmo sem afetar o resultado.

## Conclusão

> Duas ou três frases: passou? O que isso libera (próximo ensaio, integração)?

## Pendências

| O que falta | Responsável | Prazo |
|---|---|---|
| … | | |
