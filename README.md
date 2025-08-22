# Mini-Kernel Multithread com Escalonamento

## Visão Geral

Este projeto implementa um mini-kernel multithread em C que simula diferentes políticas de escalonamento de processos (FCFS, Round Robin e Prioridade Preemptiva) tanto para sistemas monoprocessador quanto multiprocessador (2 CPUs).

## Estrutura do Projeto

```
trabalho/
├── src/                    # Código fonte
│   ├── main.c             # Função principal e coordenação geral
│   ├── pcb.c              # Implementação do Process Control Block
│   ├── tcb.c              # Implementação do Task Control Block
│   ├── ready_queue.c      # Implementação da fila de prontos
│   ├── scheduler.c        # Lógica de escalonamento
│   ├── logger.c           # Sistema de logging
│   └── process_manager.c  # Gerenciamento de processos e threads
├── include/               # Headers
│   ├── pcb.h              # Definições do PCB
│   ├── tcb.h              # Definições do TCB
│   ├── ready_queue.h      # Definições da fila de prontos
│   ├── scheduler.h        # Definições do escalonador
│   ├── logger.h           # Definições do logger
│   └── process_manager.h  # Definições do gerenciador de processos
├── obj/                   # Arquivos objeto compilados
├── entradas/              # Arquivos de teste
├── saidas/                # Saídas esperadas
├── Makefile              # Sistema de build
└── README.md             # Esta documentação
```

## Compilação e Execução

### Compilação
```bash
# Versão monoprocessador
make monoprocessador

# Versão multiprocessador  
make multiprocessador

# Limpeza
make clean
```

### Execução
```bash
# Monoprocessador
./trabSO entradas/1.txt

# Multiprocessador (mesmo comando, diferente apenas na compilação)
./trabSO entradas/1.txt
```

## Decisões de Implementação

### 1. Arquitetura Modular

**Decisão**: Dividir o código em módulos especializados por funcionalidade.

**Justificativa**: 
- Facilita manutenção e debug
- Permite reutilização de código
- Melhora a legibilidade
- Separa responsabilidades claramente

**Módulos**:
- **PCB**: Gerencia estruturas de controle de processo
- **TCB**: Gerencia estruturas de controle de thread
- **Ready Queue**: Implementa fila de processos prontos
- **Scheduler**: Contém lógica de escalonamento
- **Logger**: Sistema centralizado de logging
- **Process Manager**: Gerencia criação e lifecycle de processos

### 2. Estruturas de Dados

#### Process Control Block (PCB)
```c
typedef struct {
    int pid;                    // Identificador único
    int process_len;            // Duração total (ms)
    int remaining_time;         // Tempo restante (ms)
    int priority;               // Prioridade (1=alta, 5=baixa)
    int num_threads;            // Número de threads
    int start_time;             // Tempo de chegada (ms)
    ProcessState state;         // READY, RUNNING, FINISHED
    pthread_mutex_t mutex;      // Proteção concorrente
    pthread_cond_t cv;          // Sincronização de threads
    pthread_t *thread_ids;      // IDs das threads
} PCB;
```

**Decisões**:
- Mutex individual por processo para minimizar contenção
- Variável de condição para sincronização eficiente
- Estado explícito para controle de fluxo

#### Ready Queue
```c
typedef struct {
    PCB* processes[MAX_PROCESSES];
    int front, rear, count;
    pthread_mutex_t mutex;
} ReadyQueue;
```

**Decisões**:
- Fila circular para eficiência
- Mutex próprio para operações thread-safe
- Suporte a remoção arbitrária (necessário para preempção por prioridade)

### 3. Algoritmos de Escalonamento

#### FCFS (First Come First Served)
- **Implementação**: Fila FIFO simples
- **Decisão**: Aguarda término completo do processo antes do próximo

#### Round Robin
- **Quantum**: 500ms (conforme especificação)
- **Decisão**: Preempção por tempo com recolocação na fila
- **Multiprocessador**: Rebalanceamento dinâmico após término de processos

#### Prioridade Preemptiva
- **Decisão**: Verificação contínua durante execução (a cada 50ms)
- **Preempção**: Imediata quando processo de maior prioridade chega
- **Implementação**: Busca por maior prioridade na fila durante execução

### 4. Sincronização e Concorrência

#### Estratégia de Locking
```c
// Ordem consistente para evitar deadlocks:
// 1. Scheduler mutex
// 2. Ready queue mutex  
// 3. Process mutex
```

**Decisões**:
- Ordem fixa de aquisição de locks para evitar deadlocks
- Mutexes granulares para reduzir contenção
- Uso de pthread_cond_broadcast para acordar todas as threads simultaneamente

#### Thread Lifecycle
1. **Criação**: Threads criadas quando processo chega
2. **Espera**: Bloqueiam em condition variable até estado RUNNING
3. **Execução**: Decrementam remaining_time em fatias de 50ms
4. **Finalização**: Primeira thread a detectar remaining_time <= 0 finaliza processo

### 5. Simulação de Tempo

**Decisão**: Uso de `usleep()` e `gettimeofday()` para simulação temporal precisa.

**Implementação**:
- Threads executam em fatias de 50ms
- Quantum de 500ms para Round Robin
- Tempo global baseado em `gettimeofday()`

### 6. Gerenciamento de Memória

**Estratégias**:
- Alocação dinâmica para arrays de thread IDs
- Limpeza ordenada: salvar log antes de liberar recursos
- Verificação com valgrind para garantir ausência de vazamentos

### 7. Multiprocessador vs Monoprocessador

#### Compilação Condicional
```c
#ifdef MULTIPROCESSADOR
    num_cpus = 2;
#else
    num_cpus = 1;
#endif
```

#### Diferenças de Comportamento
- **Monoprocessador**: Um processo por vez
- **Multiprocessador**: Até 2 processos simultâneos
- **Expansão de CPU**: Processo pode usar CPUs livres (exceto Round Robin)
- **Rebalanceamento**: Round Robin redistribui processos após términos

### 8. Sistema de Logging

**Decisão**: Sistema centralizado thread-safe.

**Características**:
- Buffer interno protegido por mutex
- Formato padronizado: `[ALGORITMO] Mensagem`
- Salvamento em arquivo ao final da execução



