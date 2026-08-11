# Sistema de Empréstimo de Equipamentos — Laboratório

Sistema de controle de empréstimo e devolução de equipamentos de laboratório, desenvolvido a partir do pedido da Coordenação do Laboratório (ver seção [Pedido original](#pedido-original)).

O sistema roda como um **notebook Jupyter** (`code.ipynb`), com interface em **linha de comando** (menu numerado) e persistência em **SQLite3**.

> As decisões tomadas para preencher as lacunas do pedido, as perguntas que faríamos ao cliente e os critérios de aceite estão documentados em `DECISOES.md`, entregável separado deste README.

---

## 1. Requisitos

- Python 3.10 ou superior (testado em 3.14)
- [VS Code](https://code.visualstudio.com/) com a extensão **Jupyter** (ms-toolsai.jupyter)
- Extensão **Python** do VS Code (ms-python.python)
- `sqlite3` — já incluso na biblioteca padrão do Python, nenhuma instalação extra é necessária
- `ipython`/`ipykernel` — necessário para rodar notebooks; normalmente já vem com a instalação do Jupyter

Não há dependências externas (`pip install`) — o projeto usa apenas bibliotecas padrão do Python (`sqlite3`, `datetime`, `traceback`) e o `IPython.display` do próprio Jupyter.

---

## 2. Como executar em outra máquina

1. **Clone o repositório:**
   ```bash
   git clone <https://github.com/GustavoCCol/projeto-emprestimo>
   ```

2. **Confirme que o Python está instalado:**
   ```bash
   python --version
   ```
   Se não estiver, instale o Python 3.10+ pelo [site oficial](https://www.python.org/downloads/) antes de continuar.

3. **Abra a pasta no VS Code:**
   ```bash
   code .
   ```

4. **Instale as extensões Python e Jupyter**, caso o VS Code sinalize que faltam (ele sugere automaticamente ao abrir o `.ipynb`).

5. **Abra o arquivo `code.ipynb`.**

6. **Selecione o kernel Python** no canto superior direito do notebook (deve ser o Python 3 instalado na máquina).

7. **Execute as células em ordem, de cima para baixo, uma única vez**, usando "Run All" ou `Shift+Enter` célula a célula. A ordem importa porque:
   - a primeira célula cria o banco `biblioteca_equipamentos.db` e as tabelas;
   - as células seguintes definem as funções de CRUD de aluno, equipamento e empréstimo;
   - a célula "Inserindo dados de teste" povoa o banco com 3 alunos e 5 equipamentos de exemplo;
   - a célula "Menu inicial" define e chama o menu interativo.

8. **Use o sistema pelo menu** que aparece na saída da última célula, digitando o número da opção desejada e pressionando Enter.

> **Atenção:** como o banco é o arquivo `biblioteca_equipamentos.db` salvo ao lado do notebook, rodar a célula de criação de tabelas mais de uma vez não apaga dados existentes (usa `CREATE TABLE IF NOT EXISTS`). Para reiniciar o sistema do zero, apague esse arquivo `.db` antes de rodar as células novamente.

### Reiniciando em caso de erro
Se o menu parar de responder ou o notebook travar em algum `input()`, reinicie o kernel (**Restart Kernel**, no topo do VS Code) e rode as células em ordem novamente — o sistema tem uma trava interna (`_menu_rodando`) que impede duas instâncias do menu de rodar ao mesmo tempo, mas essa trava só é redefinida com o reinício do kernel.

---

## 3. Estrutura do banco de dados

O banco é criado no arquivo `biblioteca_equipamentos.db`, com três tabelas:

**ALUNO**
| Coluna | Tipo | Descrição |
|---|---|---|
| ID | INTEGER PK | Identificador do aluno |
| NOME | VARCHAR(100) | Nome do aluno |
| BLOQUEADO_ATE | DATE | Data até quando o aluno está impedido de pegar novos equipamentos por ter devolvido algo em atraso (`NULL` se não houver bloqueio) |

**EQUIPAMENTO**
| Coluna | Tipo | Descrição |
|---|---|---|
| ID | INTEGER PK | Identificador do equipamento |
| NOME | VARCHAR(100) | Nome/descrição do equipamento |

**EMPRESTIMO**
| Coluna | Tipo | Descrição |
|---|---|---|
| ID | INTEGER PK | Identificador do empréstimo |
| ALUNO_ID | INTEGER FK → ALUNO(ID) | Aluno que pegou o equipamento |
| EQUIPAMENTO_ID | INTEGER FK → EQUIPAMENTO(ID) | Equipamento emprestado |
| DATA_EMPRESTIMO | DATE | Data em que o equipamento foi retirado |
| DATA_PREVISTA | DATE | Data prevista de devolução (sempre `DATA_EMPRESTIMO + 7 dias`) |
| DATA_DEVOLUCAO | DATE | Data em que o equipamento foi efetivamente devolvido (`NULL` enquanto em aberto) |

Chaves estrangeiras são reforçadas em tempo de execução (`PRAGMA foreign_keys = ON`), o que impede excluir um aluno ou equipamento que já tenha histórico de empréstimo.

---

## 4. Funcionalidades e regras de negócio implementadas

O menu é dividido em três blocos — Alunos, Equipamentos e Empréstimos — cobrindo o escopo mínimo pedido pela Coordenação:

**Alunos e Equipamentos:** cadastro, listagem, busca por ID, atualização e remoção (CRUD completo para as duas entidades).

**Empréstimos**, atendendo diretamente ao texto do pedido:

- *"O aluno pega o equipamento e devolve depois"* → opções **Registrar empréstimo** e **Registrar devolução**.
- *"Queremos saber o que está emprestado e para quem"* → opção **Listar empréstimos ativos**.
- *"Não queremos que os equipamentos sumam"* → um equipamento já emprestado (sem devolução registrada) não pode ser emprestado a outro aluno até ser devolvido.
- *"Aluno com pendência não pode pegar mais nada"* → um pedido de empréstimo é recusado se o aluno tiver:
  1. algum empréstimo em aberto cuja `DATA_PREVISTA` já passou (comparado com a data atual); **ou**
  2. devolvido, nos últimos 7 dias, algum equipamento após o prazo — nesse caso o aluno fica bloqueado por 7 dias a partir da data da devolução, mesmo sem nenhum empréstimo em aberto no momento.
- *"O técnico precisa de um relatório dos atrasos"* → opção **Relatório de atrasos**, listando todos os empréstimos em aberto com prazo vencido e os dias de atraso.
- Opção adicional **Listar alunos com pendência**, que mostra todos os alunos bloqueados no momento e o motivo (uma das duas situações acima).

O sistema também trata os erros básicos de operação: tentar emprestar para um aluno inexistente, tentar emprestar um equipamento inexistente, tentar devolver um empréstimo já devolvido e entradas não numéricas onde um ID é esperado — em todos os casos, o menu exibe uma mensagem de erro e volta a funcionar, sem travar.

---

## 5. Dados de teste

Ao rodar a célula "Inserindo dados de teste", o banco é populado com:

**Alunos:** Ana Souza, Bruno Lima, Carla Mendes
**Equipamentos:** Notebook Dell, Projetor Epson, Câmera Canon, Microfone Shure, Tablet Samsung

O notebook também contém, nas últimas células, uma bateria de testes manuais que simula empréstimos, atrasos e devoluções para validar as regras de bloqueio antes de abrir o menu — útil como referência de comportamento esperado do sistema.

---

## 6. Pedido original

> De: Coordenação do Laboratório
> Assunto: sistema de empréstimo
>
> Precisamos de um sistema para controlar o empréstimo dos equipamentos do laboratório. O aluno pega o equipamento e devolve depois. Queremos saber o que está emprestado e para quem, e não queremos que os equipamentos sumam. Aluno com pendência não pode pegar mais nada. O técnico precisa de um relatório dos atrasos. Tem que ser simples de usar.

As interpretações assumidas para lacunas deste pedido (como o prazo de 7 dias e a regra de bloqueio por devolução atrasada) estão detalhadas e justificadas em `DECISOES.md`.