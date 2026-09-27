# Laboratório — a mesma loja, duas arquiteturas

Estilos Arquiteturais V · Microsserviços

Nomes: Maria Letícia

Vocês vão rodar a mesma loja de dois jeitos: como um monolito (um processo, um banco) e como microsserviços (quatro processos, um banco por serviço). As duas versões têm as mesmas rotas. Ninguém precisa programar — só um experimento pede para mudar um número no código.

Requisito: Python 3.8 ou superior. Nada para instalar. No macOS/Linux use `python3`.

| Versão | Como subir | Endereço |
| --- | --- | --- |
| Monolito | `python monolito.py` | http://localhost:8000 |
| Microsserviços | `python iniciar_microsservicos.py` | http://localhost:9000 |

O objetivo é descobrir, na prática, onde os microsserviços ganham e onde eles cobram.

| Versão | Endereço | Onde ficam os dados |
| --- | --- | --- |
| Monolito | http://localhost:8000 | `dados/monolito.json` |
| Microsserviços | http://localhost:9000 (Vitrine) | `dados/estoque.json` e `dados/pedidos.json` |
| Rotas nas duas | `/produto/1`, `/comprar/1`, `/relatorio`, `/bug/estoque` |  |
| Derrubar um serviço | http://localhost:<porta>/desligar | Catálogo 9001 · Estoque 9002 · Pedidos 9003 |

**Faça:** Abra dois terminais na pasta `laboratorio`.

```bash
# terminal 1 — leva 10 s para subir
python monolito.py
```

```bash
# terminal 2 — sobe 4 processos
python iniciar_microsservicos.py
```

**Faça:** Abra duas abas no navegador, lado a lado:

- http://localhost:8000/produto/1
- http://localhost:9000/produto/1

**Deve aparecer:** O mesmo produto nas duas: Teclado mecânico, R$ 250, 5 em estoque.

### Experimento 1 · Deploy de uma promoção

O time de vendas quer 10% de desconto. Vocês vão “publicar” essa mudança nas duas versões e observar o que sai do ar enquanto isso.

**Faça — monolito**

a) Em `monolito.py`, mude `DESCONTO = 0` para `DESCONTO = 10`.

b) No terminal 1, pressione `Ctrl+C` e rode `python monolito.py` de novo.

c) Durante os 10 s de subida, tente abrir <http://localhost:8000/relatorio>.

**Faça — microsserviços**

a) Em `catalogo.py`, mude `DESCONTO = 0` para `DESCONTO = 10`.

b) Abra <http://localhost:9001/desligar> para derrubar só o Catálogo.

c) Num terceiro terminal, rode `python catalogo.py`.

d) Durante os 4 s de subida, tente <http://localhost:9000/relatorio> e <http://localhost:9000/produto/1>.

**Deve aparecer:** Depois da subida, o preço é R$ 225 nas duas.

| Aspecto | Monolito | Microsserviços |
| --- | --- | --- |
| Quanto tempo ficou fora do ar? | 10 s | 4 s |
| O que parou de funcionar? | toda a aplicação | apenas o serviço específico |
| O que continuou funcionando? | - | os outros serviços fora o do catálogo |

**Responda: Poder ou problema dos microsserviços? Por quê?** Pois, ao mesmo tempo que é bom para o usuário — porque o site continuaria funcionando normalmente —, podem ocorrer situações em que o usuário realizaria a compra enquanto o serviço de estoque está fora do ar e só depois descobriria que não havia estoque daquele produto.

### Experimento 2 · Latência

**Faça:** Num terminal livre, rode:

```bash
python comparar.py
```

**Faça:** Abra de novo `/produto/1` nas duas abas e compare o campo `tempo_interno_ms`.

| Métrica | Monolito | Microsserviços |
| --- | ---: | ---: |
| Tempo médio por página (`comparar.py`) | 7.12 ms | 29.83 ms |
| `tempo_interno_ms` | 0.025 | 21.029 |

**Responda: Aqui tudo roda no mesmo computador. O que aconteceria com essa diferença se cada serviço estivesse numa máquina diferente?** A diferença de tempo de resposta seria consideravelmente maior e, dependendo do caso, poderia ser sentida pelo usuário.

### Experimento 3 · Consistência

Na demonstração, o professor derrubou o Estoque e a loja em microsserviços continuou vendendo. Agora vocês vão ver o preço disso.

**Faça:**

a) Abra <http://localhost:9000/relatorio> e anote os números.

b) Derrube só o Estoque: <http://localhost:9002/desligar>.

c) Compre duas vezes: <http://localhost:9000/comprar/1>.

d) Religue o Estoque: `python estoque.py`.

e) Abra <http://localhost:9000/relatorio> de novo.

**Deve aparecer:** Em (c): `"aviso": "Estoque fora do ar: pedido registrado SEM baixa de estoque"`. Em (e): `"pedidos_pendentes": 2` e `"consistente": false`.

| /relatorio (microsserviços) | Antes | Depois |
| --- | ---: | ---: |
| pedidos_confirmados | 0 | 0 |
| pedidos_pendentes | 0 | 2 |
| baixas_de_estoque | 0 | 0 |
| consistente | true | false |

**Faça:** Abra a pasta `dados/`. Compare `pedidos.json` e `estoque.json`: cada banco conta uma história diferente.

**Responda:** **No monolito, o bug do Estoque derrubou a loja inteira: nenhuma venda, mas nenhum dado errado. Nos microsserviços, a loja vendeu, mas os bancos discordam. Qual dos dois a loja prefere? Quem decide isso?** Depende da estratégia da loja e não é responsabilidade dos desenvolvedores decidir isso. A loja deve decidir entre deixar o site totalmente indisponível por tempo indeterminado e não realizar vendas, ou deixar o site disponível e funcionando mesmo com algum problema e garantir vendas, lidando com os problemas de inconsistência que aparecerem depois.

### Experimento 4 · Operação

**Faça:** Faça uma compra em cada versão (`/comprar/2` nas duas) e olhe os terminais.

| Item | Monolito | Microsserviços |
| --- | --- | --- |
| Quantos processos estão rodando? | 1 | 2 |
| Quantas portas? | 1 | 2 |
| Quantos arquivos de banco? | 1 | 3 |
| Quantas linhas de log apareceram? | 1 | 2 |
| Em quantos serviços? | 1 | 2 |

**Responda: Se a compra desse errado, onde vocês procurariam o erro em cada versão?** No monolito, teria de analisar todo o código; já no microsserviços, poderíamos ir direto para o código do serviço que caiu.

## Placar da dupla

| Experimento | Quem saiu melhor? | Para microsserviços, é poder ou problema? |
| --- | --- | --- |
| Bug fatal (demonstração) | microsserviços | ambos |
| 1 · Deploy | microsserviços | poder |
| 2 · Latência | monolito | problema |
| 3 · Consistência | monolito | problema |
| 4 · Operação | microsserviços | poder |

**Para fechar: Uma loja com 3 desenvolvedores deveria usar qual das duas versões? E uma com 300? Usem o placar como argumento.** Para uma loja com 3 desenvolvedores, uma arquitetura monolito garante a simplicidade de operação, visto que eles estarão trabalhando mais em conjunto devido ao número reduzido; eles precisam de agilidade e não é ideal ter de lidar com questões de latência e consistência. Já a empresa com um número grande de devs o ideal é a arquitetura de microsserviços, pensando no ideal para escalar, e também na importância de elementos como o deploy independente e o isolamento de bug fatal, questões essenciais quando se trabalha em grandes times independentes.

Para recomeçar do zero: desliguem tudo (`Ctrl+C` nos terminais) e rodem `python resetar.py`. 


