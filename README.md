# Computação Paralela e Concorrente
### Comprovando Conceitos de Concorrência e/ou Paralelismo Usando Python e Aplicando a Metodologia de Foster

A problemática escolhida foi simples: o cálculo fatorial de um número. Para isso, foram desenvolvidas duas abordagens, uma **serial** e outra **paralela**, e ambas foram testadas individualmente para validar se o resultado retornado estava correto. A implementação não apresentou grandes dificuldades, por se tratar de um problema conceitualmente direto.

Na abordagem serial, o cálculo é realizado por meio da interação de um vetor com **N-1** elementos, em que cada multiplicação depende do resultado anterior. Apesar de eficiente para valores pequenos de **N**, o algoritmo apresenta uma limitação clara: o tempo de execução cresce linearmente conforme o número aumenta, já que todas as multiplicações são executadas de forma sequencial. Nenhuma técnica de otimização foi aplicada aqui, o código é simples e direto, sem ajustes de performance.

**O principal gargalo** dessa versão está justamente nessa sequência de operações, além do fato de o **Python ser uma linguagem interpretada**, o que torna as **multiplicações menos eficientes** do que em linguagens compiladas. Outro ponto é que, conforme os números crescem, os inteiros ocupam mais espaço e exigem mais processamento.

```python
def serial_factorial(num):
    pre_result = 1
    for i in range(2, num + 1):
        pre_result *= i
    return pre_result
```

De modo geral, a versão **serial utiliza poucos recursos**: apenas um núcleo da CPU e uma quantidade mínima de memória (já que há apenas uma variável acumuladora). O tempo de **CPU, entretanto, cresce proporcionalmente a N**, e para **valores muito grandes o desempenho se degrada perceptivelmente**. Assim, enquanto o método **serial é adequado para números menores, ele não escala bem, abrindo espaço para uma solução paralela**.

Na abordagem paralela, foi utilizada a **biblioteca multiprocessing do Python**, que permite **distribuir o processamento em vários núcleos da CPU**. Mais especificamente, foi utilizado o recurso **Pool**, que cria um conjunto de processos executados em paralelo. Com isso, cada processo calcula uma parte do fatorial e envia o resultado parcial ao processo principal, onde ocorre a multiplicação final. O método **cpu_count()** foi utilizado para identificar automaticamente quantos núcleos estavam disponíveis e, assim, ajustar a quantidade de processos.

```python
def parallel_factorial(num: int) -> int:
    num_procs = cpu_count()
    blocks = split_blocks(num, num_procs)

    with Pool(num_procs) as pool:
        results = pool.map(partial_product, blocks)

    result_final = 1
    for r in results:
        result_final *= r
    return result_final
```

Como o **Pool gerencia internamente a criação, execução e finalização dos processos**, não houve necessidade de aplicar técnicas manuais de sincronização. A única **etapa explícita de controle é o uso do pool.map()**, que garante que todos os processos terminem antes da combinação dos resultados.

Para adaptar o código serial para a abordagem paralela, o algoritmo foi reestruturado em três etapas principais. Primeiramente, ocorreu a **divisão do problema**, com a criação da função **split_blocks**, responsável por particionar o intervalo [1, N] em blocos menores, permitindo que cada processo recebesse um trecho do cálculo. Em seguida, foi desenvolvida a função **partial_product**, que **realiza o cálculo independente do produto em cada intervalo**. Por fim, após o término das execuções paralelas, os **resultados parciais são combinados no processo principal por meio de multiplicações sucessivas**. Essa reestruturação modular tornou possível dividir, executar e integrar as partes do problema de forma clara e eficiente, seguindo o padrão típico da programação paralela.

```python
def partial_product(range_of_values):
    inital_value, final_value = range_of_values
    result_parcial = 1
    for i in range(inital_value, final_value + 1):
        result_parcial *= i
    return result_parcial


def split_blocks(num, num_blocks):
    size = num // num_blocks
    blocks = []
    inital_value = 1
    for i in range(num_blocks):
        final_value = inital_value + size - 1 if i < num_blocks - 1 else num
        blocks.append((inital_value, final_value))
        inital_value = final_value + 1
    return blocks
```

Os resultados observados com essa abordagem foram expressivos. **Houve redução significativa do tempo de execução para valores grandes de N**, já que múltiplos núcleos puderam processar blocos simultaneamente. Além disso, o uso da **CPU tornou-se mais eficiente**, aproveitando todos os núcleos disponíveis. O método também se mostrou **escalável**, pois quanto maior o número de núcleos, mais blocos podem ser processados em paralelo. Em cargas computacionais mais pesadas, o ganho de desempenho é notável. No entanto, para **valores pequenos, o custo de criação e sincronização dos processos pode superar o benefício**, tornando a execução serial mais vantajosa.

Por fim, a implementação seguiu os princípios da metodologia de Foster, respeitando suas quatro etapas fundamentais. O **particionamento** foi aplicado na função **split_blocks**, que dividiu o problema em blocos menores; a **comunicação** ocorreu de forma implícita por meio do **pool.map**, responsável pela distribuição e coleta dos resultados; a **aglomeração** se deu na **multiplicação final dos resultados parciais** dentro do processo principal; e o **mapeamento** consistiu na distribuição das tarefas entre os núcleos disponíveis, utilizando **cpu_count()** e **Pool**. Essa estrutura sistemática garantiu um paralelismo eficiente e um código bem organizado, evidenciando na prática os benefícios da computação paralela em relação à execução serial.

Por fim, vamos analisar as métricas de tempo das execuções, **a análise foi feita com o valor inicial de 200 até 1.000.000, em intervalos de 100, onde foi feito um encerramento da operação do cálculo fatorial quando o tempo de processo passasse de 3 segundos, assim temos valores no intervalo de 200 a 330.300**. O tempo de execução foi medido utilizando a função **time() da biblioteca padrão do Python (time)**, capturando o tempo antes e depois da execução de cada método de cálculo fatorial. Assim, o tempo total corresponde à diferença entre esses dois instantes.

```python
def run_factorial(method_factorial, num):
    init_time = time.time()
    result = method_factorial(num)
    final_time = time.time()

    return init_time, result, final_time


get_time = lambda init_time, final_time: final_time - init_time


def print_run_factorial(method_factorial, num):
    init_time, result, final_time = run_factorial(method_factorial, num)

    if result > 9999999999:
        sci_result = Decimal(result).normalize()
        print(f"Fatorial de {num} é {sci_result:.3E}")
    else:
        print(f"Fatorial de {num} é {result}")
    print(f"Tempo de processamento: {get_time(init_time, final_time):.4f} segundos")
```

Na execução serial, observamos que o tempo de processamento cresce de forma **não linear conforme o valor de entrada (N) aumenta**, refletindo o comportamento esperado do algoritmo fatorial, que envolve **N-1** multiplicações sequenciais. No gráfico **“Execução Serial”**, a curva azul mostra um crescimento acentuado, **ultrapassando 3 segundos para valores próximos de 100.000**. Esse comportamento confirma a **natureza O(n) do algoritmo**, onde o tempo de execução aumenta proporcionalmente ao número de multiplicações.

![Gráfico “Execução Serial”](https://github.com/Gbrl454/foster-paralela-concorrente/blob/main/Execução%20Serial.png)

Na execução paralela, o comportamento observado é significativamente diferente. O gráfico **“Execução Paralela"** (linha verde) mostra que o tempo cresce de forma muito mais lenta, **permanecendo em torno de 0,05 a 0,50 segundos mesmo para valores de entrada próximos de 100.000**. Isso representa uma **redução de cerca de 6 a 8 vezes no tempo de execução** em relação à abordagem serial, evidenciando um ganho expressivo de desempenho com o uso de múltiplos núcleos. Cada processo parcial é executado de **forma independente dentro de um pool de processos gerenciado pela biblioteca multiprocessing**, e o tempo total medido corresponde ao intervalo completo, desde o início da criação dos processos até a combinação dos resultados parciais.

![Gráfico “Execução Paralela”](https://github.com/Gbrl454/foster-paralela-concorrente/blob/main/Execução%20Paralela.png)

Ao observar o gráfico combinado **“Execução Serial vs Paralela”**, nota-se com clareza a diferença de inclinação entre as curvas. **A linha azul (serial) cresce de forma exponencial**, enquanto a **linha verde (paralela) mantém uma evolução suave e quase linear** até o final da execução. Isso mostra que o paralelismo consegue distribuir a carga de multiplicações entre múltiplos núcleos, reduzindo o tempo total de execução. Entretanto, é possível observar pequenas flutuações na curva paralela, especialmente para valores menores de **N**, que se devem ao overhead da criação e sincronização dos processos. Esse custo inicial faz com que, **para entradas pequenas, a versão serial ainda seja mais eficiente**. Contudo, **conforme o número cresce, o custo do overhead torna-se irrelevante frente ao ganho obtido pela execução simultânea**.

![Gráfico “Execução Serial vs Paralela”](https://github.com/Gbrl454/foster-paralela-concorrente/blob/main/Execução%20Serial%20vs%20Paralela.png)
