# Documentação Técnica do Projeto - Processador de Consultas SQL

## 📋 Sumário

1. [Visão Geral do Projeto](#visão-geral-do-projeto)
2. [Arquitetura do Sistema](#arquitetura-do-sistema)
3. [HU1 - Entrada e Validação da Consulta](#hu1---entrada-e-validação-da-consulta)
4. [HU2 - Conversão para Álgebra Relacional](#hu2---conversão-para-álgebra-relacional)
5. [HU3 - Construção do Grafo de Operadores](#hu3---construção-do-grafo-de-operadores)
6. [HU4 - Otimização da Consulta](#hu4---otimização-da-consulta)
7. [HU5 - Plano de Execução](#hu5---plano-de-execução)
8. [Modelo de Dados](#modelo-de-dados)
9. [Exemplos de Uso](#exemplos-de-uso)
10. [Conclusão](#conclusão)

---

## Visão Geral do Projeto

O **Processador de Consultas SQL** é um sistema acadêmico desenvolvido para a disciplina de Banco de Dados que implementa as principais etapas de processamento de consultas SQL encontradas em sistemas de gerenciamento de banco de dados (SGBDs).

### Objetivos

- Validar consultas SQL quanto à sintaxe e existência de tabelas/atributos
- Converter consultas SQL em expressões de álgebra relacional
- Construir grafos de operadores para visualização da estratégia de execução
- Aplicar heurísticas de otimização para melhorar o desempenho
- Gerar planos de execução ordenados

### Tecnologias Utilizadas

- **Python 3.7+**: Linguagem principal do projeto
- **Tkinter**: Interface gráfica nativa do Python
- **Graphviz**: Geração de grafos visuais
- **Pillow (PIL)**: Manipulação de imagens

---

## Arquitetura do Sistema

O sistema está organizado em módulos especializados, cada um responsável por uma etapa específica do processamento:

```
main.py                  → Interface gráfica e orquestração
sql_parser.py            → Parsing e validação SQL (HU1)
algebra_converter.py     → Conversão para álgebra relacional (HU2)
algebra_expressions.py   → Classes de expressões algébricas
query_optimizer.py       → Otimização de consultas (HU4)
graph_builder.py         → Construção do grafo (HU3)
execution_planner.py     → Plano de execução (HU5)
metadata.py              → Esquema do banco de dados
```

### Fluxo de Processamento

```
1. Entrada SQL (Interface)
   ↓
2. Parsing e Validação (SQLParser)
   ↓
3. Conversão Álgebra (AlgebraConverter)
   ↓
4. Otimização (QueryOptimizer)
   ↓
5. Construção Grafo (GraphBuilder)
   ↓
6. Plano Execução (ExecutionPlanner)
   ↓
7. Exibição Resultados (Interface)
```

---

## HU1 - Entrada e Validação da Consulta

### Descrição

Como usuário do sistema, quero digitar uma consulta SQL na interface gráfica para que o sistema valide sintaxe, tabelas e atributos existentes.

### Critérios de Aceitação Implementados

✅ Interface gráfica com campo de entrada da consulta  
✅ Parser valida comandos SQL básicos (SELECT, FROM, WHERE, JOIN, ON)  
✅ Operadores válidos (=, >, <, <=, >=, <>, AND, ( ))  
✅ Verificação de existência de tabelas e atributos  
✅ Ignora diferença entre maiúsculas e minúsculas  
✅ Ignora repetições de espaços em branco  
✅ Suporta múltiplos JOINs (0, 1, ..., N)

### Implementação

#### 1. Interface Gráfica (`main.py`)

A interface foi implementada usando Tkinter com uma área de texto scrollável para entrada de consultas:

```python
def create_widgets(self):
    # Área de entrada SQL
    ttk.Label(main_frame, text="Consulta SQL:", font=("Arial", 12, "bold")).grid(
        row=0, column=0, sticky=tk.W, pady=(0, 5)
    )
    
    self.sql_input = scrolledtext.ScrolledText(main_frame, height=6, width=80)
    self.sql_input.grid(row=1, column=0, sticky=(tk.W, tk.E), pady=(0, 10))
    
    # Botão processar
    ttk.Button(
        main_frame, text="Processar Consulta", command=self.process_query
    ).grid(row=2, column=0, pady=(0, 10))
```

**Justificativa**: O `scrolledtext.ScrolledText` permite entrada de consultas longas e complexas com múltiplos JOINs, atendendo ao requisito de interface amigável.

#### 2. Normalização da Consulta (`sql_parser.py`)

O parser normaliza a consulta removendo espaços extras e preservando a estrutura:

```python
def _normalize_query(self, query):
    """Remove espaços extras e normaliza a consulta"""
    query = re.sub(r"\s+", " ", query)  # Substitui múltiplos espaços por um
    query = query.strip()
    return query
```

**Justificativa**: Atende ao requisito de ignorar repetições de espaços em branco, facilitando o parsing.

#### 3. Validação de Sintaxe Básica

```python
def _validate_basic_syntax(self, query):
    """Valida estrutura básica SELECT ... FROM ..."""
    query_upper = query.upper()
    
    if "SELECT" not in query_upper or "FROM" not in query_upper:
        return False
    
    # SELECT deve vir antes de FROM
    select_pos = query_upper.index("SELECT")
    from_pos = query_upper.index("FROM")
    
    if select_pos >= from_pos:
        return False
    
    return True
```

**Justificativa**: Valida a estrutura mínima necessária de uma consulta SQL válida. O uso de `.upper()` implementa a funcionalidade case-insensitive.

#### 4. Extração de Componentes da Consulta

```python
def _extract_query_components(self, query):
    """Extrai os componentes SELECT, FROM, JOIN, WHERE"""
    parsed = {
        "select": [],
        "from": [],
        "joins": [],
        "where": None,
        "original": query,
    }
    
    # Extrair SELECT
    select_start = query_upper.index("SELECT") + 6
    from_start = query_upper.index("FROM")
    select_clause = query[select_start:from_start].strip()
    parsed["select"] = [col.strip() for col in select_clause.split(",")]
    
    # Processar FROM e JOINs
    self._parse_from_joins(from_clause, parsed)
    
    return parsed
```

**Justificativa**: Separa a consulta em componentes estruturados para facilitar validação e conversão posterior.

#### 5. Suporte a Múltiplos JOINs

```python
def _parse_from_joins(self, from_clause, parsed):
    """Processa FROM e múltiplos JOINs"""
    from_clause = re.sub(r"\bFROM\b", "", from_clause, flags=re.IGNORECASE).strip()
    
    # Separar por JOIN
    parts = re.split(r"\bJOIN\b", from_clause, flags=re.IGNORECASE)
    
    # Primeira parte é a tabela FROM
    first_table = parts[0].strip()
    parsed["from"].append(first_table)
    
    # Processar cada JOIN
    for i in range(1, len(parts)):
        join_part = parts[i].strip()
        
        # Extrair tabela e condição ON
        on_match = re.search(r"\bON\b", join_part, re.IGNORECASE)
        if on_match:
            table = join_part[: on_match.start()].strip()
            condition = join_part[on_match.start() + 2 :].strip()
            parsed["joins"].append({"table": table, "condition": condition})
```

**Justificativa**: Usa expressões regulares com flags `re.IGNORECASE` para detectar JOINs independentemente da capitalização, suportando N junções conforme especificado.

#### 6. Validação de Tabelas

```python
def _validate_tables(self, parsed):
    """Valida existência de todas as tabelas mencionadas"""
    # Validar tabela FROM
    for table in parsed["from"]:
        if not table_exists(table):
            return False, f"Tabela '{table}' não existe no esquema"
    
    # Validar tabelas dos JOINs
    for join in parsed["joins"]:
        table = join["table"]
        if not table_exists(table):
            return False, f"Tabela '{table}' não existe no esquema"
    
    return True, "Tabelas válidas"
```

**Justificativa**: Garante que todas as tabelas mencionadas existem no esquema definido em `metadata.py`.

#### 7. Validação de Colunas

```python
def _validate_columns(self, parsed):
    """Valida existência de todas as colunas nas tabelas corretas"""
    all_tables = parsed["from"] + [join["table"] for join in parsed["joins"]]
    
    # Validar colunas do SELECT
    for col_expr in parsed["select"]:
        if "." in col_expr:
            # Formato: tabela.coluna
            parts = col_expr.split(".")
            if len(parts) == 2:
                table, column = parts[0].strip(), parts[1].strip()
                if not table_exists(table):
                    return False, f"Tabela '{table}' não encontrada"
                if not column_exists_in_table(table, column):
                    return False, f"Coluna '{column}' não existe na tabela '{table}'"
    
    # Validar colunas nas condições JOIN
    for join in parsed["joins"]:
        columns_in_condition = re.findall(r"(\w+\.\w+)", join["condition"])
        for col_expr in columns_in_condition:
            parts = col_expr.split(".")
            table, column = parts[0], parts[1]
            if not column_exists_in_table(table, column):
                return False, f"Coluna '{column}' não existe na tabela '{table}'"
    
    return True, "Colunas válidas"
```

**Justificativa**: Verifica se cada coluna existe na tabela correspondente, incluindo colunas no SELECT, JOIN ON e WHERE.

#### 8. Validação de Operadores

```python
def _validate_operators(self, parsed):
    """Valida uso de operadores permitidos"""
    if parsed["where"]:
        where_clause = parsed["where"].upper()
        
        # Verificar operadores
        for op in ["<>", "<=", ">=", "=", "<", ">", "AND"]:
            if op in where_clause:
                if op not in self.valid_operators:
                    return False, f"Operador '{op}' não é válido"
    
    return True, "Operadores válidos"
```

**Justificativa**: Garante que apenas operadores especificados no enunciado sejam utilizados (=, >, <, <=, >=, <>, AND).

### Metadados do Banco de Dados (`metadata.py`)

O esquema do banco de dados é definido de forma estruturada:

```python
SCHEMA = {
    "Cliente": {
        "columns": ["idCliente", "Nome", "Email", "Nascimento", "Senha", 
                    "TipoCliente_idTipoCliente", "DataRegistro"],
        "primary_key": "idCliente",
        "foreign_keys": {"TipoCliente_idTipoCliente": ("TipoCliente", "idTipoCliente")},
    },
    "Pedido": {
        "columns": ["idPedido", "Status_idStatus", "DataPedido", 
                    "ValorTotalPedido", "Cliente_idCliente"],
        "primary_key": "idPedido",
        "foreign_keys": {
            "Status_idStatus": ("Status", "idStatus"),
            "Cliente_idCliente": ("Cliente", "idCliente"),
        },
    },
    # ... outras tabelas
}

def table_exists(table_name):
    """Verifica se tabela existe (case-insensitive)"""
    return normalize_table_name(table_name) in SCHEMA

def column_exists_in_table(table_name, column_name):
    """Verifica se coluna existe na tabela (case-insensitive)"""
    columns = get_table_columns(table_name)
    if columns:
        column_name_lower = column_name.lower()
        return any(col.lower() == column_name_lower for col in columns)
    return False
```

**Justificativa**: Define o modelo de dados completo do sistema de e-commerce, incluindo chaves primárias e estrangeiras, permitindo validação robusta.

### Exemplo de Uso

```sql
-- Consulta válida com múltiplos JOINs
SELECT Cliente.Nome, Pedido.ValorTotalPedido, Status.Descricao
FROM Cliente
JOIN Pedido ON Cliente.idCliente = Pedido.Cliente_idCliente
JOIN Status ON Pedido.Status_idStatus = Status.idStatus
WHERE Pedido.ValorTotalPedido > 100 AND Status.idStatus <> 5
```

**Resultado**: Consulta validada com sucesso, pronta para conversão.

---

## HU2 - Conversão para Álgebra Relacional

### Descrição

Como aluno, quero que minha consulta SQL seja convertida em uma expressão de álgebra relacional, para compreender a representação teórica.

### Critérios de Aceitação Implementados

✅ Exibir a consulta equivalente em álgebra relacional na interface gráfica  
✅ A conversão preserva operadores e condições  
✅ Representação inclui seleção (σ), projeção (π) e junções (⋈)

### Implementação

#### 1. Classes de Expressões Algébricas (`algebra_expressions.py`)

Base para todas as expressões:

```python
class AlgebraExpression:
    """Classe base para expressões de álgebra relacional"""
    def to_string(self, indent=0):
        raise NotImplementedError
    
    def get_tables(self):
        raise NotImplementedError
```

**Projeção (π)**:

```python
class Projection(AlgebraExpression):
    """Projeção (π) - seleciona colunas específicas"""
    
    def __init__(self, attributes, child):
        self.attributes = attributes  # Lista de atributos a projetar
        self.child = child            # Expressão filha
    
    def to_string(self, indent=0):
        attrs = ", ".join(self.attributes)
        indent_str = "  " * indent
        child_str = self.child.to_string(indent + 1)
        return f"{indent_str}π {attrs} (\n{child_str}\n{indent_str})"
    
    def get_tables(self):
        return self.child.get_tables()
```

**Justificativa**: Representa a operação de projeção que seleciona apenas as colunas especificadas no SELECT.

**Seleção (σ)**:

```python
class Selection(AlgebraExpression):
    """Seleção (σ) - filtra tuplas com base em condição"""
    
    def __init__(self, condition, child):
        self.condition = condition  # Condição de filtro
        self.child = child          # Expressão filha
    
    def to_string(self, indent=0):
        # Converte AND para símbolo de conjunção (^)
        cond = self.condition.replace(" and ", " ^ ").replace(" AND ", " ^ ")
        indent_str = "  " * indent
        child_str = self.child.to_string(indent + 1)
        return f"{indent_str}σ {cond} (\n{child_str}\n{indent_str})"
    
    def get_tables(self):
        return self.child.get_tables()
```

**Justificativa**: Representa a operação de seleção que filtra tuplas de acordo com a cláusula WHERE.

**Junção (⋈)**:

```python
class Join(AlgebraExpression):
    """Junção (⋈) - combina duas relações"""
    
    def __init__(self, condition, left, right):
        self.condition = condition  # Condição de junção
        self.left = left            # Expressão esquerda
        self.right = right          # Expressão direita
    
    def to_string(self, indent=0):
        indent_str = "  " * indent
        left_str = self.left.to_string(indent + 1)
        right_str = self.right.to_string(indent + 1)
        return f"{indent_str}(\n{left_str}\n{indent_str}  ⋈ {self.condition}\n{right_str}\n{indent_str})"
    
    def get_tables(self):
        return self.left.get_tables() + self.right.get_tables()
```

**Justificativa**: Representa a operação de junção natural ou equi-join entre duas relações.

**Tabela**:

```python
class Table(AlgebraExpression):
    """Tabela (relação base) - nó folha"""
    
    def __init__(self, name):
        self.name = name
    
    def to_string(self, indent=0):
        indent_str = "  " * indent
        return f"{indent_str}{self.name}"
    
    def get_tables(self):
        return [self.name]
```

**Justificativa**: Representa uma tabela do banco de dados (nó folha da árvore de expressão).

#### 2. Conversor de Álgebra (`algebra_converter.py`)

```python
class AlgebraConverter:
    def convert(self, parsed_query):
        """Converte consulta SQL parseada para álgebra relacional"""
        
        # 1. Coletar todas as tabelas (FROM + JOINs)
        tables = parsed_query["from"] + [
            join["table"] for join in parsed_query["joins"]
        ]
        
        # 2. Construir junções (ou tabela única)
        if len(tables) == 1:
            result = Table(tables[0])
        else:
            result = self._build_joins(parsed_query)
        
        # 3. Aplicar seleção (WHERE)
        if parsed_query["where"]:
            result = Selection(parsed_query["where"], result)
        
        # 4. Aplicar projeção (SELECT)
        result = Projection(parsed_query["select"], result)
        
        return result
```

**Justificativa**: Constrói a árvore de expressão algébrica seguindo a ordem lógica: junções → seleção → projeção.

**Construção de múltiplas junções**:

```python
def _build_joins(self, parsed_query):
    """Constrói árvore de junções para múltiplos JOINs"""
    
    # Começar com a primeira tabela
    result = Table(parsed_query["from"][0])
    
    # Adicionar cada junção sequencialmente
    for join in parsed_query["joins"]:
        right_table = Table(join["table"])
        condition = join["condition"]
        result = Join(condition, result, right_table)
    
    return result
```

**Justificativa**: Constrói uma árvore balanceada à esquerda para múltiplos JOINs, compatível com a ordem de execução típica de SGBDs.

#### 3. Exibição na Interface (`main.py`)

```python
def process_query(self):
    # ... (após parsing)
    
    # HU2: Conversão para Álgebra Relacional
    algebra_expr = self.algebra_converter.convert(parsed_query)
    self.algebra_text.insert(1.0, "PASSO 1 - HEURÍSTICA DE JUNÇÃO:\n\n")
    self.algebra_text.insert(tk.END, algebra_expr.to_string())
```

**Justificativa**: Mostra a expressão algébrica formatada com indentação para facilitar compreensão.

### Exemplo de Conversão

**Entrada SQL**:
```sql
SELECT Cliente.Nome, Pedido.DataPedido
FROM Cliente
JOIN Pedido ON Cliente.idCliente = Pedido.Cliente_idCliente
WHERE Cliente.idCliente > 100
```

**Saída em Álgebra Relacional**:
```
π Cliente.Nome, Pedido.DataPedido (
  σ Cliente.idCliente > 100 (
    (
      Cliente
      ⋈ Cliente.idCliente = Pedido.Cliente_idCliente
      Pedido
    )
  )
)
```

**Interpretação**: 
1. Junção entre Cliente e Pedido
2. Seleção dos clientes com ID > 100
3. Projeção dos atributos Nome e DataPedido

---

## HU3 - Construção do Grafo de Operadores

### Descrição

Como aluno, quero que o sistema construa um grafo de operadores, para visualizar a estratégia de execução da consulta.

### Critérios de Aceitação Implementados

✅ O grafo é gerado em memória e exibido na interface  
✅ Cada nó representa operadores  
✅ Arestas representam fluxo de resultados intermediários  
✅ As folhas representam as tabelas  
✅ A raiz representa a última projeção  
✅ O grafo representa a estratégia de execução

### Implementação

#### 1. Estrutura do Nó do Grafo (`graph_builder.py`)

```python
class GraphNode:
    """Nó do grafo de operadores"""
    
    def __init__(self, operator, details, children=None):
        self.operator = operator    # Tipo: Projection, Selection, Join, Table
        self.details = details      # Detalhes específicos do operador
        self.children = children if children else []  # Filhos (operadores abaixo)
        self.id = None             # ID único para renderização
    
    def add_child(self, child):
        self.children.append(child)
```

**Justificativa**: Representa cada operador como um nó com tipo, detalhes e referências aos operadores dependentes.

#### 2. Grafo de Operadores

```python
class OperatorGraph:
    """Grafo completo de operadores"""
    
    def __init__(self, root):
        self.root = root           # Nó raiz (última operação - geralmente projeção)
        self.node_counter = 0      # Contador para IDs únicos
```

**Justificativa**: Encapsula a árvore de operadores com a raiz representando o resultado final.

#### 3. Construtor do Grafo

```python
class GraphBuilder:
    def build_graph(self, algebra_expr):
        """Constrói grafo a partir da expressão algébrica"""
        root = self._build_node(algebra_expr)
        return OperatorGraph(root)
    
    def _build_node(self, expr):
        """Constrói nó recursivamente"""
        
        if isinstance(expr, Projection):
            attrs = ", ".join(expr.attributes)
            node = GraphNode("Projeção (π)", attrs)
            child = self._build_node(expr.child)
            node.add_child(child)
            return node
        
        elif isinstance(expr, Selection):
            node = GraphNode("Seleção (σ)", expr.condition)
            child = self._build_node(expr.child)
            node.add_child(child)
            return node
        
        elif isinstance(expr, Join):
            node = GraphNode("Junção (⋈)", expr.condition)
            left_child = self._build_node(expr.left)
            right_child = self._build_node(expr.right)
            node.add_child(left_child)
            node.add_child(right_child)
            return node
        
        elif isinstance(expr, Table):
            node = GraphNode("Tabela", expr.name)
            return node
        
        else:
            node = GraphNode("Desconhecido", str(expr))
            return node
```

**Justificativa**: Constrói recursivamente o grafo visitando cada nó da expressão algébrica. As folhas são sempre tabelas, e a raiz é sempre uma projeção.

#### 4. Renderização Visual com Graphviz

```python
def render_graphviz(self, filename="query_graph", view=False):
    """Renderiza grafo usando Graphviz"""
    
    dot = graphviz.Digraph(comment="Query Execution Graph")
    dot.attr(rankdir="TB")  # Top to Bottom (raiz no topo, folhas na base)
    dot.attr("node", shape="box", style="rounded,filled", fontname="Arial")
    
    self.node_counter = 0
    self._add_nodes_to_graphviz(dot, self.root)
    
    # Renderizar como PNG
    output_path = f"/tmp/{filename}"
    dot.render(output_path, format="png", cleanup=True, view=view)
    
    return f"{output_path}.png"
```

**Justificativa**: Usa Graphviz para gerar visualização profissional do grafo com direção de cima para baixo.

**Adição recursiva de nós**:

```python
def _add_nodes_to_graphviz(self, dot, node, parent_id=None):
    """Adiciona nós recursivamente ao grafo Graphviz"""
    
    # Gerar ID único
    node.id = f"node_{self.node_counter}"
    self.node_counter += 1
    
    # Determinar cor e label baseado no tipo de operador
    if node.operator == "Projeção (π)":
        color = "#E3F2FD"  # Azul claro
        label = f"π\n{node.details}"
    elif node.operator == "Seleção (σ)":
        color = "#FFF3E0"  # Laranja claro
        label = f"σ\n{node.details}"
    elif node.operator == "Junção (⋈)":
        color = "#F3E5F5"  # Roxo claro
        label = f"⋈\n{node.details}"
    elif node.operator == "Tabela":
        color = "#E8F5E9"  # Verde claro
        label = node.details
    else:
        color = "#F5F5F5"  # Cinza claro
        label = f"{node.operator}\n{node.details}"
    
    # Adicionar nó
    dot.node(node.id, label, fillcolor=color)
    
    # Conectar ao pai se existir
    if parent_id:
        dot.edge(parent_id, node.id)
    
    # Processar filhos
    for child in node.children:
        self._add_nodes_to_graphviz(dot, child, node.id)
```

**Justificativa**: Cores diferentes para cada tipo de operador facilitam identificação visual. As arestas representam o fluxo de dados dos operadores filhos para o pai.

#### 5. Integração com Interface (`main.py`)

```python
def process_query(self):
    # ... (após otimização)
    
    # HU3: Construção do Grafo
    graph = self.graph_builder.build_graph(optimized_algebra)
    
    # Gerar grafo visual
    try:
        image_path = graph.render_graphviz(filename="grafo_consulta", view=False)
        
        # Carregar e exibir imagem
        from PIL import Image, ImageTk
        
        img = Image.open(image_path)
        
        # Redimensionar se necessário
        max_width = 750
        max_height = 550
        img.thumbnail((max_width, max_height), Image.Resampling.LANCZOS)
        
        photo = ImageTk.PhotoImage(img)
        
        # Criar label com imagem
        label = ttk.Label(self.graph_inner_frame, image=photo)
        label.image = photo  # Manter referência
        label.pack(padx=10, pady=10)
        
    except Exception as e:
        error_label = ttk.Label(
            self.graph_inner_frame,
            text=f"Erro ao gerar grafo visual: {e}",
            foreground="red",
        )
        error_label.pack(padx=10, pady=10)
```

**Justificativa**: Exibe o grafo na interface com tratamento de erros caso Graphviz não esteja instalado.

### Características do Grafo

- **Direção**: Top-Down (raiz no topo, folhas na base)
- **Raiz**: Última operação (geralmente Projeção)
- **Folhas**: Tabelas base
- **Arestas**: Fluxo de resultados intermediários
- **Cores**: Diferenciação visual por tipo de operador

### Exemplo Visual

Para a consulta:
```sql
SELECT Cliente.Nome, Pedido.DataPedido
FROM Cliente
JOIN Pedido ON Cliente.idCliente = Pedido.Cliente_idCliente
WHERE Cliente.idCliente > 100
```

O grafo mostra:
```
          [π: Nome, DataPedido]  (Azul)
                   ↓
          [σ: idCliente > 100]  (Laranja)
                   ↓
           [⋈: idCliente = ...]  (Roxo)
              ↙        ↘
        [Cliente]    [Pedido]   (Verde)
```

---

## HU4 - Otimização da Consulta

### Descrição

Como aluno, quero que a álgebra relacional seja otimizada conforme heurísticas, para reduzir o custo de execução.

### Critérios de Aceitação Implementados

✅ Seleções que reduzem tuplas primeiro  
✅ Projeções que reduzem atributos na sequência  
✅ Seleções e junções mais restritivas primeiro  
✅ Evitar produto cartesiano  
✅ Exibir o grafo otimizado

### Implementação

O otimizador aplica heurísticas em 3 passos conforme especificado:

#### Passo 1: Heurística de Junção

Esta é a álgebra inicial convertida diretamente do SQL, já construída pelo `AlgebraConverter`. O sistema usa JOINs explícitos com condições ON, evitando produtos cartesianos.

```python
# Em algebra_converter.py
def _build_joins(self, parsed_query):
    """Passo 1: Constrói junções com condições explícitas"""
    result = Table(parsed_query["from"][0])
    
    for join in parsed_query["joins"]:
        right_table = Table(join["table"])
        condition = join["condition"]
        # Junção com condição explícita (não produto cartesiano)
        result = Join(condition, result, right_table)
    
    return result
```

**Justificativa**: Ao exigir condições ON em todos os JOINs, evitamos produtos cartesianos desde o início.

#### Passo 2: Heurística de Redução de Tuplas (Push Selections)

Move seleções (filtros) o mais próximo possível das tabelas base, reduzindo o número de tuplas antes de operações custosas como junções.

```python
class QueryOptimizer:
    def _apply_tuple_reduction(self, expr):
        """Passo 2: Empurra seleções para as tabelas"""
        
        if isinstance(expr, Projection):
            if isinstance(expr.child, Selection):
                # Empurrar seleção através das junções
                new_child = self._push_selection_to_tables(
                    expr.child.condition, expr.child.child
                )
                return Projection(expr.attributes, new_child)
        return expr
    
    def _push_selection_to_tables(self, condition, join_expr):
        """Separa condições e aplica a cada tabela"""
        
        # Separar condições por AND
        conditions = re.split(r"\s+and\s+", condition, flags=re.IGNORECASE)
        
        # Classificar condições por tabela
        table_conditions = {}
        for cond in conditions:
            # Extrair tabela da condição (ex: Cliente.id > 300)
            match = re.match(r"(\w+)\.", cond)
            if match:
                table = match.group(1)
                if table not in table_conditions:
                    table_conditions[table] = []
                table_conditions[table].append(cond)
        
        # Aplicar seleções recursivamente na árvore
        return self._apply_selections_to_tree(join_expr, table_conditions)
```

**Justificativa**: Reduz o número de tuplas o mais cedo possível, diminuindo o custo das junções subsequentes.

**Aplicação recursiva**:

```python
def _apply_selections_to_tree(self, expr, table_conditions):
    """Aplica seleções na árvore de junções"""
    
    if isinstance(expr, Join):
        # Processar recursivamente ambos os lados
        left = self._apply_selections_to_tree(expr.left, table_conditions)
        right = self._apply_selections_to_tree(expr.right, table_conditions)
        return Join(expr.condition, left, right)
    
    elif isinstance(expr, Table):
        # Aplicar seleção se houver condições para esta tabela
        table_name = expr.name
        for key in table_conditions:
            if key.lower() == table_name.lower():
                conds = table_conditions[key]
                if conds:
                    condition_str = " AND ".join(conds)
                    return Selection(condition_str, expr)
        return expr
    
    return expr
```

**Justificativa**: Percorre a árvore de junções e aplica seleções diretamente sobre as tabelas relevantes.

#### Passo 3: Heurística de Redução de Campos (Push Projections)

Adiciona projeções intermediárias após seleções para reduzir o número de atributos carregados, diminuindo o uso de memória e I/O.

```python
def _apply_field_reduction(self, expr):
    """Passo 3: Adiciona projeções após seleções para reduzir campos"""
    
    if isinstance(expr, Projection):
        final_attrs = expr.attributes
        new_child = self._add_projections_after_selections(
            expr.child, set(final_attrs)
        )
        return Projection(final_attrs, new_child)
    return expr

def _add_projections_after_selections(self, expr, needed_attrs):
    """Adiciona projeções intermediárias"""
    
    if isinstance(expr, Join):
        # Coletar atributos necessários incluindo os do join
        join_attrs = self._extract_attributes_from_condition(expr.condition)
        all_needed = needed_attrs.union(set(join_attrs))
        
        # Determinar quais atributos vêm de cada lado
        left_tables = self._get_tables_from_expr(expr.left)
        right_tables = self._get_tables_from_expr(expr.right)
        
        left_attrs = []
        right_attrs = []
        
        for attr in all_needed:
            table = attr.split(".")[0] if "." in attr else ""
            if table.lower() in [t.lower() for t in left_tables]:
                left_attrs.append(attr)
            elif table.lower() in [t.lower() for t in right_tables]:
                right_attrs.append(attr)
        
        # Processar recursivamente
        left_child = self._add_projections_after_selections(
            expr.left, set(left_attrs)
        )
        right_child = self._add_projections_after_selections(
            expr.right, set(right_attrs)
        )
        
        return Join(expr.condition, left_child, right_child)
    
    elif isinstance(expr, Selection):
        # Coletar atributos da condição também
        cond_attrs = self._extract_attributes_from_condition(expr.condition)
        all_attrs = needed_attrs.union(set(cond_attrs))
        
        # Construir: Projeção(Seleção(Tabela))
        result = Selection(expr.condition, expr.child)
        
        # Adicionar projeção APÓS a seleção
        if all_attrs:
            return Projection(sorted(list(all_attrs)), result)
        return result
    
    elif isinstance(expr, Table):
        # Tabela sem seleção: adicionar projeção direto
        if needed_attrs:
            return Projection(sorted(list(needed_attrs)), expr)
        return expr
    
    return expr
```

**Justificativa**: Reduz a quantidade de dados manipulados em cada operação, carregando apenas os atributos necessários.

**Funções auxiliares**:

```python
def _extract_attributes_from_condition(self, condition):
    """Extrai atributos mencionados na condição"""
    return re.findall(r"(\w+\.\w+)", condition)

def _get_tables_from_expr(self, expr):
    """Obtém todas as tabelas de uma expressão"""
    if isinstance(expr, Table):
        return [expr.name]
    elif isinstance(expr, Join):
        return self._get_tables_from_expr(expr.left) + self._get_tables_from_expr(expr.right)
    elif isinstance(expr, Selection):
        return self._get_tables_from_expr(expr.child)
    elif isinstance(expr, Projection):
        return self._get_tables_from_expr(expr.child)
    return []
```

### Exemplo de Otimização

**Entrada SQL**:
```sql
SELECT Cliente.Nome, Pedido.DataPedido
FROM Cliente
JOIN Pedido ON Cliente.idCliente = Pedido.Cliente_idCliente
WHERE Cliente.idCliente > 100 AND Pedido.ValorTotalPedido > 200
```

**Passo 1 - Álgebra Inicial** (heurística de junção):
```
π Cliente.Nome, Pedido.DataPedido (
  σ Cliente.idCliente > 100 ^ Pedido.ValorTotalPedido > 200 (
    (
      Cliente
      ⋈ Cliente.idCliente = Pedido.Cliente_idCliente
      Pedido
    )
  )
)
```

**Passo 2 - Redução de Tuplas** (push selections):
```
π Cliente.Nome, Pedido.DataPedido (
  (
    σ Cliente.idCliente > 100 (
      Cliente
    )
    ⋈ Cliente.idCliente = Pedido.Cliente_idCliente
    σ Pedido.ValorTotalPedido > 200 (
      Pedido
    )
  )
)
```

**Passo 3 - Redução de Campos** (push projections):
```
π Cliente.Nome, Pedido.DataPedido (
  (
    π Cliente.idCliente, Cliente.Nome (
      σ Cliente.idCliente > 100 (
        Cliente
      )
    )
    ⋈ Cliente.idCliente = Pedido.Cliente_idCliente
    π Pedido.Cliente_idCliente, Pedido.DataPedido (
      σ Pedido.ValorTotalPedido > 200 (
        Pedido
      )
    )
  )
)
```

**Benefícios da Otimização**:
1. **Redução de I/O**: Filtros aplicados antes da junção
2. **Menos memória**: Apenas atributos necessários são carregados
3. **Junção mais eficiente**: Menos tuplas e atributos para combinar

---

## HU5 - Plano de Execução

### Descrição

Como aluno, quero visualizar a ordem de execução da consulta, para compreender como o banco executaria passo a passo.

### Critérios de Aceitação Implementados

✅ Exibir ordem de execução (plano de execução ordenado)  
✅ Listar operações na ordem correta  
✅ Execução segue ordem definida pelo grafo otimizado

### Implementação

#### 1. Estrutura de Passo de Execução (`execution_planner.py`)

```python
class ExecutionStep:
    """Representa um passo no plano de execução"""
    
    def __init__(self, step_number, operation, details, dependencies=None):
        self.step_number = step_number      # Número do passo
        self.operation = operation          # Tipo de operação
        self.details = details              # Detalhes da operação
        self.dependencies = dependencies if dependencies else []  # Passos dependentes
    
    def __str__(self):
        deps = (
            f" (depende de: {', '.join(map(str, self.dependencies))})"
            if self.dependencies
            else ""
        )
        return f"Passo {self.step_number}: {self.operation} - {self.details}{deps}"
```

**Justificativa**: Cada passo tem número sequencial, descrição e dependências, permitindo compreensão clara da ordem de execução.

#### 2. Plano de Execução

```python
class ExecutionPlan:
    """Plano de execução completo"""
    
    def __init__(self):
        self.steps = []
    
    def add_step(self, step):
        self.steps.append(step)
    
    def to_string(self):
        result = "PLANO DE EXECUÇÃO\n"
        result += "=" * 80 + "\n\n"
        result += "Ordem de execução (sequencial, das folhas para a raiz):\n\n"
        
        for step in self.steps:
            result += str(step) + "\n"
        
        return result
```

**Justificativa**: Organiza os passos de forma linear e compreensível.

#### 3. Geração do Plano (Percurso Pós-Ordem)

```python
class ExecutionPlanner:
    def create_plan(self, operator_graph):
        """Cria plano de execução a partir do grafo"""
        plan = ExecutionPlan()
        self.step_counter = 0
        
        # Percurso pós-ordem (folhas primeiro, raiz por último)
        self._traverse_postorder(operator_graph.root, plan)
        
        return plan
```

**Justificativa**: Percurso pós-ordem garante que operações dependentes sejam executadas antes de operações que dependem delas (bottom-up).

**Percurso recursivo**:

```python
def _traverse_postorder(self, node, plan, parent_steps=None):
    """Percorre grafo em pós-ordem, criando passos de execução"""
    
    if parent_steps is None:
        parent_steps = []
    
    current_dependencies = []
    
    # Processar filhos primeiro (recursão)
    for child in node.children:
        child_step = self._traverse_postorder(child, plan, current_dependencies)
        if child_step:
            current_dependencies.append(child_step)
    
    self.step_counter += 1
    
    # Criar passo baseado no tipo de operador
    if node.operator == "Tabela":
        step = ExecutionStep(
            self.step_counter,
            "Scan de Tabela",
            f"Ler dados da tabela '{node.details}'",
            [],
        )
    
    elif node.operator == "Seleção (σ)":
        step = ExecutionStep(
            self.step_counter,
            "Aplicar Seleção",
            f"Filtrar tuplas usando condição: {node.details}",
            current_dependencies,
        )
    
    elif node.operator == "Projeção (π)":
        step = ExecutionStep(
            self.step_counter,
            "Aplicar Projeção",
            f"Selecionar colunas: {node.details}",
            current_dependencies,
        )
    
    elif node.operator == "Junção (⋈)":
        step = ExecutionStep(
            self.step_counter,
            "Executar Junção",
            f"Juntar tabelas usando condição: {node.details}",
            current_dependencies,
        )
    
    else:
        step = ExecutionStep(
            self.step_counter, node.operator, node.details, current_dependencies
        )
    
    plan.add_step(step)
    return self.step_counter
```

**Justificativa**: 
- **Pós-ordem**: Visita filhos antes do nó atual, garantindo ordem correta de execução
- **Dependências**: Cada passo conhece os passos anteriores dos quais depende
- **Descrições claras**: Cada tipo de operador tem descrição específica

#### 4. Integração com Interface

```python
def process_query(self):
    # ... (após construção do grafo)
    
    # HU5: Plano de Execução
    execution_plan = self.execution_planner.create_plan(graph)
    self.execution_text.insert(1.0, execution_plan.to_string())
```

### Exemplo de Plano de Execução

Para a consulta otimizada:
```sql
SELECT Cliente.Nome, Pedido.DataPedido
FROM Cliente
JOIN Pedido ON Cliente.idCliente = Pedido.Cliente_idCliente
WHERE Cliente.idCliente > 100 AND Pedido.ValorTotalPedido > 200
```

**Plano de Execução Gerado**:
```
PLANO DE EXECUÇÃO
================================================================================

Ordem de execução (sequencial, das folhas para a raiz):

Passo 1: Scan de Tabela - Ler dados da tabela 'Cliente'
Passo 2: Aplicar Seleção - Filtrar tuplas usando condição: Cliente.idCliente > 100 (depende de: 1)
Passo 3: Aplicar Projeção - Selecionar colunas: Cliente.idCliente, Cliente.Nome (depende de: 2)
Passo 4: Scan de Tabela - Ler dados da tabela 'Pedido'
Passo 5: Aplicar Seleção - Filtrar tuplas usando condição: Pedido.ValorTotalPedido > 200 (depende de: 4)
Passo 6: Aplicar Projeção - Selecionar colunas: Pedido.Cliente_idCliente, Pedido.DataPedido (depende de: 5)
Passo 7: Executar Junção - Juntar tabelas usando condição: Cliente.idCliente = Pedido.Cliente_idCliente (depende de: 3, 6)
Passo 8: Aplicar Projeção - Selecionar colunas: Cliente.Nome, Pedido.DataPedido (depende de: 7)
```

**Interpretação**:
1. **Passos 1-3**: Processa tabela Cliente (scan → filtro → projeção)
2. **Passos 4-6**: Processa tabela Pedido (scan → filtro → projeção)
3. **Passo 7**: Junta resultados intermediários
4. **Passo 8**: Projeta colunas finais

**Benefícios**:
- Execução paralela possível (passos 1-3 e 4-6 são independentes)
- Redução de dados em cada etapa
- Ordem lógica e otimizada

---

## Modelo de Dados

O sistema trabalha com um esquema de banco de dados de e-commerce com as seguintes tabelas:

### Diagrama de Relacionamentos

```
Categoria ←── Produto ──→ Pedido_has_Produto ←── Pedido ←── Cliente ←── Endereco
                                                     ↓             ↑         ↑
                                                   Status    TipoCliente  TipoEndereco
                                                                   ↑
                                                              Telefone
```

### Tabelas Principais

1. **Cliente**: Informações de clientes
   - Colunas: idCliente (PK), Nome, Email, Nascimento, Senha, TipoCliente_idTipoCliente (FK), DataRegistro

2. **Pedido**: Pedidos realizados
   - Colunas: idPedido (PK), Status_idStatus (FK), DataPedido, ValorTotalPedido, Cliente_idCliente (FK)

3. **Produto**: Produtos disponíveis
   - Colunas: idProduto (PK), Nome, Descricao, Preco, QuantEstoque, Categoria_idCategoria (FK)

4. **Pedido_has_Produto**: Relacionamento N:N entre Pedidos e Produtos
   - Colunas: idPedidoProduto (PK), Pedido_idPedido (FK), Produto_idProduto (FK), Quantidade, PrecoUnitario

5. **Status**: Status dos pedidos
   - Colunas: idStatus (PK), Descricao

6. **Categoria**: Categorias de produtos
   - Colunas: idCategoria (PK), Descricao

7. **TipoCliente**: Tipos de clientes
   - Colunas: idTipoCliente (PK), Descricao

8. **Endereco**: Endereços dos clientes
   - Colunas: idEndereco (PK), EnderecoPadrao, Logradouro, Numero, Complemento, Bairro, Cidade, UF, CEP, TipoEndereco_idTipoEndereco (FK), Cliente_idCliente (FK)

9. **TipoEndereco**: Tipos de endereços
   - Colunas: idTipoEndereco (PK), Descricao

10. **Telefone**: Telefones dos clientes
    - Colunas: Numero (PK), Cliente_idCliente (FK)

---

## Exemplos de Uso

### Exemplo 1: Consulta Simples

**SQL**:
```sql
SELECT Cliente.Nome, Cliente.Email
FROM Cliente
WHERE Cliente.idCliente > 100
```

**Álgebra Relacional**:
```
π Cliente.Nome, Cliente.Email (
  σ Cliente.idCliente > 100 (
    Cliente
  )
)
```

**Plano de Execução**:
```
Passo 1: Scan de Tabela - Ler dados da tabela 'Cliente'
Passo 2: Aplicar Seleção - Filtrar tuplas usando condição: Cliente.idCliente > 100
Passo 3: Aplicar Projeção - Selecionar colunas: Cliente.Nome, Cliente.Email
```

---

### Exemplo 2: Consulta com 1 JOIN

**SQL**:
```sql
SELECT Cliente.Nome, Pedido.DataPedido, Pedido.ValorTotalPedido
FROM Cliente
JOIN Pedido ON Cliente.idCliente = Pedido.Cliente_idCliente
WHERE Cliente.idCliente > 50
```

**Álgebra Relacional Otimizada**:
```
π Cliente.Nome, Pedido.DataPedido, Pedido.ValorTotalPedido (
  (
    π Cliente.idCliente, Cliente.Nome (
      σ Cliente.idCliente > 50 (
        Cliente
      )
    )
    ⋈ Cliente.idCliente = Pedido.Cliente_idCliente
    π Pedido.Cliente_idCliente, Pedido.DataPedido, Pedido.ValorTotalPedido (
      Pedido
    )
  )
)
```

**Plano de Execução**:
```
Passo 1: Scan de Tabela - Ler dados da tabela 'Cliente'
Passo 2: Aplicar Seleção - Filtrar tuplas usando condição: Cliente.idCliente > 50
Passo 3: Aplicar Projeção - Selecionar colunas: Cliente.idCliente, Cliente.Nome
Passo 4: Scan de Tabela - Ler dados da tabela 'Pedido'
Passo 5: Aplicar Projeção - Selecionar colunas: Pedido.Cliente_idCliente, Pedido.DataPedido, Pedido.ValorTotalPedido
Passo 6: Executar Junção - Juntar tabelas usando condição: Cliente.idCliente = Pedido.Cliente_idCliente
Passo 7: Aplicar Projeção - Selecionar colunas: Cliente.Nome, Pedido.DataPedido, Pedido.ValorTotalPedido
```

---

### Exemplo 3: Consulta com Múltiplos JOINs e Condições

**SQL**:
```sql
SELECT Cliente.Nome, Pedido.ValorTotalPedido, Status.Descricao
FROM Cliente
JOIN Pedido ON Cliente.idCliente = Pedido.Cliente_idCliente
JOIN Status ON Pedido.Status_idStatus = Status.idStatus
WHERE Pedido.ValorTotalPedido > 100 AND Status.idStatus <> 5
```

**Álgebra Relacional Otimizada**: Aplica todas as heurísticas
1. Seleções movidas para as tabelas base
2. Projeções intermediárias reduzem atributos
3. Junções executadas com menos dados

**Plano de Execução**: Executa operações na ordem otimizada, minimizando o custo computacional.

---

## Conclusão

### Resultados Alcançados

O projeto implementou com sucesso todas as 5 histórias de usuário especificadas:

1. ✅ **HU1**: Sistema valida consultas SQL com suporte a múltiplos JOINs, case-insensitive, com validação completa de tabelas e atributos
2. ✅ **HU2**: Conversão para álgebra relacional preserva todas as operações (σ, π, ⋈)
3. ✅ **HU3**: Grafo visual com cores diferenciadas mostra estratégia de execução
4. ✅ **HU4**: Otimização em 3 passos aplica heurísticas reconhecidas (redução de tuplas e campos)
5. ✅ **HU5**: Plano de execução detalhado com dependências explícitas

### Arquitetura Modular

A organização em módulos especializados facilita:
- **Manutenção**: Cada módulo tem responsabilidade única
- **Extensibilidade**: Novos operadores ou heurísticas podem ser adicionados
- **Testabilidade**: Componentes podem ser testados isoladamente

### Heurísticas de Otimização

O sistema implementa as principais heurísticas ensinadas em cursos de banco de dados:
1. **Push Selections**: Filtros aplicados o mais cedo possível
2. **Push Projections**: Apenas atributos necessários são carregados
3. **Join Ordering**: Junções explícitas evitam produtos cartesianos
4. **Field Reduction**: Minimiza uso de memória e I/O

### Interface Intuitiva

A interface gráfica com abas separadas permite:
- Entrada fácil de consultas
- Visualização clara da álgebra relacional (3 passos)
- Grafo visual colorido
- Plano de execução detalhado

### Valor Educacional

O sistema é uma excelente ferramenta de aprendizado porque:
- Mostra como SGBDs processam consultas internamente
- Visualiza conceitos abstratos (álgebra relacional)
- Demonstra impacto das otimizações
- Permite experimentação com diferentes consultas

### Possíveis Extensões Futuras

1. **Mais operadores SQL**: GROUP BY, ORDER BY, agregações
2. **Custos estimados**: Calcular custo de cada operação
3. **Múltiplos planos**: Comparar diferentes estratégias
4. **Estatísticas**: Usar estatísticas das tabelas para otimizações baseadas em custos
5. **Índices**: Considerar índices disponíveis no plano de execução

---

**Desenvolvido para a disciplina de Gerenciamento de Banco de Dados**  
**Universidade de Fortaleza - 2024**
