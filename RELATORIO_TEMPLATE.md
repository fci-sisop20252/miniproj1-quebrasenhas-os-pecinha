# Relatório: Mini-Projeto 1 - Quebra-Senhas Paralelo

**Aluno(s):** Artur Campi (10436740), Lucas Franco (10439830), Marcelo Kenzo ()
---

## 1. Estratégia de Paralelização


**Como você dividiu o espaço de busca entre os workers?**

O coordinator divide o espaço total de combinações de forma equilibrada entre os workers. Se sobrarem combinações (resto da divisão), elas são distribuídas aos primeiros workers, garantindo que todos tenham cargas quase iguais e o trabalho termine de forma mais justa e rápida

**Código relevante:** Cole aqui a parte do coordinator.c onde você calcula a divisão:
```c

long long passwords_per_worker = total_space / num_workers;
long long remaining = total_space % num_workers;

long long current_start = 0;
for (int i = 0; i < num_workers; i++) {
    long long workload = passwords_per_worker;
    if (i < remaining) workload++;
    long long end_index = current_start + workload - 1;

    index_to_password(current_start, charset, charset_len, password_len, start_password);
    index_to_password(end_index,   charset, charset_len, password_len, end_password);

    current_start = end_index + 1;
}

```

---

## 2. Implementação das System Calls

**Descreva como você usou fork(), execl() e wait() no coordinator:**

No coordinator, cada worker é criado com fork(): o processo pai continua a coordenar, enquanto o processo filho usa execl() para substituir sua imagem pelo programa worker, recebendo como argumentos o hash alvo, a senha inicial, a senha final, o charset, o tamanho da senha e o ID do worker. Assim, cada filho passa a executar apenas o seu intervalo de senhas. O processo pai guarda os PIDs dos filhos para controle e, ao final, utiliza wait() em um loop para aguardar a conclusão de todos, garantindo que não fiquem processos zumbis e que o coordenador só finalize quando todos os workers terminarem

**Código do fork/exec:**
```c

long long current_start = 0;
for (int i = 0; i < num_workers; i++) {
    long long workload = passwords_per_worker;
    if (i < remaining) workload++;
    long long end_index = current_start + workload - 1;

    char start_password[password_len + 1];
    char end_password[password_len + 1];
    index_to_password(current_start, charset, charset_len, password_len, start_password);
    index_to_password(end_index,   charset, charset_len, password_len, end_password);

    pid_t pid = fork();
    if (pid == 0) {
        char worker_id_str[10];
        char password_len_str[10];
        sprintf(worker_id_str, "%d", i);
        sprintf(password_len_str, "%d", password_len);

        execl("./worker", "worker", target_hash, start_password, end_password, 
              charset, password_len_str, worker_id_str, NULL);

        perror("Erro no execl");
        exit(1);
    } else {
        workers[i] = pid;
        printf("Worker %d (PID: %d) iniciado - Intervalo: [%s, %s]\n", i, pid, start_password, end_password);
    }
    current_start = end_index + 1;
}

```

---

## 3. Comunicação Entre Processos

**Como você garantiu que apenas um worker escrevesse o resultado?**

Implementamos a escrita de forma atômica usando open() com O_CREAT | O_EXCL. Assim, apenas o primeiro worker consegue criar o arquivo; os demais falham se ele já existir. Isso evita condição de corrida e garante que só um resultado seja gravado

**Como o coordinator consegue ler o resultado?**

Após o término dos workers, o coordinator abre password_found.txt, faz o parse do formato worker_id:senha com fscanf, recalcula o MD5 da senha com md5_string() e compara com o hash alvo para validar o resultado

---

## 4. Análise de Performance
Complete a tabela com tempos reais de execução:
O speedup é o tempo do teste com 1 worker dividido pelo tempo com 4 workers.

| Teste | 1 Worker | 2 Workers | 4 Workers | Speedup (4w) |
|-------|----------|-----------|-----------|--------------|
| Hash: 202cb962ac59075b964b07152d234b70<br>Charset: "0123456789"<br>Tamanho: 3<br>Senha: "123" | 0.00s | 0.00s | 0.00s |Indefinido (tempo muito pequeno)|
| Hash: 5d41402abc4b2a76b9719d911017c592<br>Charset: "abcdefghijklmnopqrstuvwxyz"<br>Tamanho: 5<br>Senha: "hello" | 4.00s | 8.00s | 2.00s | 2.00 |

**O speedup foi linear? Por quê?**

Dobrar o número de workers não dobrou a velocidade. No teste, 2 workers chegaram a ser mais lentos e 4 workers deram apenas 2× de speedup em relação a 1. Isso acontece por causa do overhead de criação/sincronização de processos (fork/exec/wait), da checagem periódica do arquivo de resultado (parada não imediata), do desbalanceamento da divisão em blocos contíguos e da competição por CPU. Esses fatores impedem que o ganho seja linear

---

## 5. Desafios e Aprendizados
**Qual foi o maior desafio técnico que você enfrentou?**

O maior desafio foi o incremento de senha. Resolvi tratando a senha como um contador em base variável, percorrendo de trás para frente e fazendo “vai-um” quando estourava o charset, garantindo todas as combinações corretas

---

## Comandos de Teste Utilizados

```bash
# Teste básico
./coordinator "900150983cd24fb0d6963f7d28e17f72" 3 "abc" 2

# Teste de performance
time ./coordinator "202cb962ac59075b964b07152d234b70" 3 "0123456789" 1
time ./coordinator "202cb962ac59075b964b07152d234b70" 3 "0123456789" 4

# Teste com senha maior
time ./coordinator "5d41402abc4b2a76b9719d911017c592" 5 "abcdefghijklmnopqrstuvwxyz" 4
```
---

**Checklist de Entrega:**
- [✓] Código compila sem erros
- [✓] Todos os TODOs foram implementados
- [✓] Testes passam no `./tests/simple_test.sh`
- [✓] Relatório preenchido