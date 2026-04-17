# Analista de Dados Power BI



***

# ✅ Projeto “Order-to-Delivery” (Complexo/Realista) — Blueprint completo

## 1) O que você vai entregar (o que fica perfeito no portfólio)

Um relatório Power BI com:

*   **Resumo executivo**: Receita, Pedidos, Ticket médio, %SLA, Atraso médio, OTIF (opcional), Nota média
*   **Logística**: atrasos por transportadora/rota/UF, gargalo (expedição vs transporte)
*   **Cliente / Qualidade**: avaliação vs atraso, devoluções/cancelamentos (se tiver)

E por trás disso: um **modelo estrela bem montado**, com **múltiplas tabelas fato** (bem real em empresas).

***

# 2) Modelo Estrela (Star Schema) recomendado

## ⭐ Grão (grain) principal — escolha “profissional”

**FatoPedidoItem (Order Line)**: 1 linha = 1 item do pedido

> Esse grão é o mais realista (permite analisar por produto/categoria e não só por pedido).

### ✅ Dimensões (Dim tables)

Crie estas dimensões (o “esqueleto” do modelo):

1.  **DimData** (uma só, mas usada como *role-playing* via medidas)
    *   Data, Ano, Mês, Nome do mês, Semana, Dia da semana, etc.

2.  **DimProduto**
    *   Produto, Categoria, Marca (se tiver), SKU

3.  **DimCliente**
    *   Cliente, Segmento (se tiver), cidade/UF (ou chave para DimLocal)

4.  **DimVendedor / DimLoja** (dependendo do dataset)
    *   Seller/Loja/CD

5.  **DimTransportadora**
    *   Transportadora, modal (se tiver)

6.  **DimLocal** (recomendado — centraliza geografia)
    *   Cidade, UF, Região  
        (pode ter “tipo local”: Cliente, Loja, CD)

7.  **DimStatusPedido** *(opcional, mas fica bem corporativo)*
    *   Status: entregue/cancelado/devolvido/em trânsito…

> **Dica de arquitetura**: use DimLocal e relacione DimCliente e DimVendedor a ela via chaves (ou traga cidade/UF direto para cada Dim se preferir simplificar).

***

## ✅ Tabelas Fato (Fact tables) — versão “empresa”

Você pode ter 2 ou 3 fatos, dependendo dos dados que escolher:

### (A) **FatoPedidoItem** *(principal)*

Campos típicos:

*   OrderID, OrderItemID (ou linha)
*   ProductID, CustomerID, SellerID
*   **OrderDateKey** (data do pedido)
*   Quantidade, Preço, Desconto (se tiver), Receita, Custo (se tiver), Frete rateado (opcional)

### (B) **FatoEntrega** *(logística)*

Grão comum:

*   1 linha por pedido **ou** 1 linha por entrega (se tiver eventos)
    Campos:
*   OrderID
*   CarrierID
*   **ShipDate**, **DeliveryDate**, **PromisedDate** (se tiver)
*   StatusEntrega, AtrasoDias, LeadTimeDias
*   Indicadores: OnTime (0/1), Delivered (0/1)

### (C) **FatoPagamento / FatoFinanceiro** *(opcional, dá muito “brilho”)*

*   OrderID, PaymentDate
*   Método (cartão/boleto/pix), parcelas
*   Valor pago, taxas (se tiver)

### (D) **FatoAvaliação / Satisfação** *(se tiver)*

*   OrderID, ReviewDate
*   Nota (1–5), comentário (não precisa carregar texto inteiro)

> **Por que múltiplos fatos?**  
> Porque isso é o que acontece em empresas: vendas ≠ logística ≠ financeiro ≠ qualidade. Quem sabe modelar isso ganha muitos pontos.

***

# 3) Datas “em papéis diferentes” (sem bagunçar relacionamento)

Você vai ter várias datas relevantes:

*   Data do pedido (OrderDate)
*   Data de envio (ShipDate)
*   Data de entrega (DeliveryDate)
*   Data prometida (PromisedDate) — se existir
*   Data do pagamento (PaymentDate)
*   Data da avaliação (ReviewDate)

### ✅ Abordagem recomendada (bem usada no mercado)

*   Relacione **DimData → FatoPedidoItem** pela **OrderDate** (ativa)
*   Para as demais datas, você tem 2 opções “profissionais”:
    1.  **Criar DimData duplicada por papel** (DimDataEntrega, DimDataEnvio…)  
        **OU**
    2.  **Manter uma DimData** e usar **medidas com USERELATIONSHIP** para ativar relações inativas.

Se você está começando, a opção (1) é mais simples de entender. A opção (2) é mais elegante e comum em modelos avançados.

***

# 4) Power Query (ETL) — o que fazer para ficar “real”

Checklist de transformações “de empresa”:

*   Tipos corretos (datas, números, texto)
*   Remover duplicatas por chave
*   Criar chaves (se não existirem): ProductKey, CustomerKey…
*   Normalizar categorias (ex.: “Eletrônicos”, “eletronicos”, “Eletrônico” → “Eletrônicos”)
*   Tratar nulos (principalmente datas)
*   Criar colunas úteis:
    *   `LeadTime = DeliveryDate - OrderDate`
    *   `TempoExpedicao = ShipDate - OrderDate`
    *   `TempoTransporte = DeliveryDate - ShipDate`
    *   `Atraso = DeliveryDate - PromisedDate` (se existir)

> **Dica:** Se o dataset não tiver PromisedDate, você pode **criar uma “data prometida estimada”** baseada em regra de negócio (ex.: “pedido + 5 dias úteis”). Isso mostra criatividade, mas deixe claro que é estimativa.

***

# 5) Medidas DAX (pacote “contratável”)

Abaixo um kit de medidas que já deixa seu projeto muito profissional:

## 📌 Comerciais

```DAX
Receita = SUM ( FatoPedidoItem[Receita] )

Pedidos = DISTINCTCOUNT ( FatoPedidoItem[OrderID] )

Itens = COUNTROWS ( FatoPedidoItem )

Ticket Medio = DIVIDE ( [Receita], [Pedidos] )
```

## 📌 Logística

(Considerando FatoEntrega com flags e datas)

```DAX
Entregas = DISTINCTCOUNT ( FatoEntrega[OrderID] )

% Entregue = 
DIVIDE(
    CALCULATE([Entregas], FatoEntrega[StatusEntrega] = "Entregue"),
    [Entregas]
)

Atraso Medio (dias) = AVERAGE ( FatoEntrega[AtrasoDias] )

Lead Time Medio (dias) = AVERAGE ( FatoEntrega[LeadTimeDias] )

% On Time =
DIVIDE(
    CALCULATE([Entregas], FatoEntrega[OnTime] = 1),
    [Entregas]
)
```

## 📌 Gargalo (expedição vs transporte)

```DAX
Tempo Expedicao Medio = AVERAGE ( FatoEntrega[TempoExpedicaoDias] )

Tempo Transporte Medio = AVERAGE ( FatoEntrega[TempoTransporteDias] )

% Atraso por Expedicao =
DIVIDE(
    CALCULATE([Entregas], FatoEntrega[TempoExpedicaoDias] > FatoEntrega[TempoTransporteDias]),
    [Entregas]
)
```

## 📌 Qualidade / Satisfação (se houver avaliação)

```DAX
Nota Media = AVERAGE ( FatoAvaliacao[Nota] )

Nota Media OnTime =
CALCULATE(
    [Nota Media],
    TREATAS( VALUES(FatoEntrega[OrderID]), FatoAvaliacao[OrderID] ),
    FatoEntrega[OnTime] = 1
)

Nota Media Atrasado =
CALCULATE(
    [Nota Media],
    TREATAS( VALUES(FatoEntrega[OrderID]), FatoAvaliacao[OrderID] ),
    FatoEntrega[OnTime] = 0
)
```

> **Por que isso é bom?**  
> Porque mostra que você sabe **cruzar fatos** (Entrega ↔ Avaliação) — algo muito comum em BI corporativo.

***

# 6) Layout sugerido (3 páginas “perfeitas para portfólio”)

## Página 1 — **Executive Summary**

*   Receita, Pedidos, Ticket Médio
*   % On Time, Atraso Médio, Lead Time Médio
*   Gráfico de tendência mensal
*   Top categorias/UF
*   Um card “Insight do mês” (texto)

## Página 2 — **Logística & SLA**

*   Ranking transportadoras por %OnTime
*   Heatmap/Mapa por UF (opcional)
*   Distribuição por faixa de atraso (0, 1–3, 4–7, 8+)
*   Gargalo: expedição vs transporte

## Página 3 — **Cliente & Qualidade**

*   Nota média por UF/categoria
*   Nota vs atraso (faixas)
*   Devoluções/cancelamentos (se tiver)

***

# 7) Validações “de empresa” (isso te destaca)

Antes de finalizar, faça checagens rápidas:

*   Total de pedidos na FatoPedidoItem = total de pedidos na FatoEntrega (ou explique diferença)
*   Datas sem entrega (em aberto) não devem entrar em métricas de entrega
*   Categorias com poucos dados: avalie filtros/limiares
*   Performance: evite colunas calculadas pesadas; prefira **medidas**

***

# ✅ Próximo passo: escolher a fonte de dados “complexa e real”

Para montar esse modelo realista, você tem 2 caminhos muito bons:

1.  **Dataset público de e-commerce com entregas e avaliações** (mais “vida real”)
2.  **Dataset corporativo de varejo (tipo Contoso/AdventureWorks)** e você cria a parte logística (entrega/prazos) como tabela adicional

✅ **Uma única pergunta para eu te indicar o caminho mais eficiente:**
Você quer que o projeto fique mais **“Brasil e-commerce real”** (com entregas/avaliações prontas) ou mais **“corporativo clássico”** (varejo/ERP e você modela logística junto)?
