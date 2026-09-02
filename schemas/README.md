# Schemas de Entidade — MVP

Toda entidade do mundo é uma pasta com um arquivo `.md` (frontmatter + body). O
World Validator (`server/validator.py`) rejeita frontmatter que não cumpra o
contrato abaixo. Tipos do MVP: `location`, `character`, `route`, `memory`,
`intention`, `item`, `object`. (`island`, `event`: fase posterior — `region` já
não é: é só uma `location` grande contendo outras, ver spec 035 abaixo.)

O **body** é sempre a narrativa diegética — o que se vê, sente e ouve. Nunca use
termos de sistema no body de forma exposta ao player.

## Autoria (como criar/editar o mundo)

O mundo é só pasta + arquivo `.md` — você edita direto no editor de texto, sem servidor,
banco nem rede (SC-007). A estrutura de pastas É a topologia:

```
world/
  <regiao-id>/location.md                        # uma location GRANDE contendo outras
    <cidade-id>/location.md                       # uma location dentro dela — mesmo tipo
      <location-id>/location.md                    # um lugar (taverna, praça, ...)
        <quarto-id>/location.md                      # sub-location dentro do lugar
          <character-id>/character.md                 # um personagem NAQUELE lugar
            memories/<memoria-id>.md                  # memórias do personagem
            <item-id>/item.md                         # item do personagem
              <item-aninhado-id>/item.md               # item dentro de item
        <object-id>/object.md                        # objeto fixo NAQUELE lugar (não portável)
          <item-id>/item.md                          # item aninhado (ex.: loot de um baú)
  routes/<route-id>/route.md                     # uma rota entre locations (quaisquer duas)
    <object-id>/object.md                        # objeto ancorado na própria rota
```

- **Criar** uma entidade = criar a pasta e o `.md` com o frontmatter do tipo (campos abaixo).
- **Editar** = alterar o `.md`. O `id` deve ser único e estável (é a âncora do personagem).
- **Mover** um personagem de lugar = mover a **pasta** dele para dentro de outra `location/`.
  Itens e memórias vão junto (estão dentro da pasta). Na próxima consulta ele está lá.
- **Validação na leitura**: todo `.md` é checado pelo World Validator. Um arquivo inválido é
  **ignorado no jogo** (não corrompe o mundo) e reportado com motivo legível no terminal do
  server (no boot) e em `GET /api/world/health`. Corrija o motivo e recarregue.
- Arquivos nunca são apagados pelo sistema; memórias vencidas viram `state: expired`.

## location — `location.md`
Obrigatórios: `type`, `id`, `name`, `size` (regra do tamanho — ver `size` abaixo).
Opcionais: `entry_point` (sub-estrutura padrão de chegada, hoje não lido pelo Motor —
campo morto), `origin`.

- `size`: régua `PP < P < M < G < XG < XXG < XXXG < XXXXG < XXXXXG`. É dela que sai
  o tempo de ATRAVESSAR o lugar quando alguém está só de passagem (spec 012); os
  degraus do topo (`XXXXG`, `XXXXXG`) existem justamente para escala geográfica —
  uma cidade ou uma região usam esse fim da régua.
- **Uma location PODE conter outra location aninhada na própria pasta** (spec 035):
  região contém cidade, cidade contém taverna, taverna contém quarto — sem
  precisar de campo `parent` algum, porque a pasta já é a hierarquia. Cada
  `location.md` é a fronteira de uma cena própria: "onde alguém está" é sempre o
  **pai direto** da pasta dele, nunca um ancestral mais de cima — um personagem
  dentro do quarto não está "na taverna" para fins de jogo, só narrativamente (o
  corpo do quarto pode mencionar que fica em cima da taverna). Isso vale também
  para itens: um item dentro de uma sub-location NUNCA aparece na cena da
  location que a contém, mesmo que ela esteja fisicamente "por cima". Chegar à
  cidade por uma rota longa te deixa na location da CIDADE (o portão/praça —
  pense nela como o hub); dali, rotas curtas levam a cada prédio específico.

## character — `character.md`
Obrigatórios: `type`, `id`, `name`, `controlled_by`, `attributes`, `skills`, `status`.
- `attributes`: mapa com os 6 atributos inteiros `STR, DEX, CON, INT, WIS, CHA`.
  Modifier = `floor((valor - 10) / 2)`.
- `weight_kg` (opcional, spec 010): peso do CORPO em kg (número > 0). Ausente ⇒
  **70** (adulto típico), então nenhum personagem já escrito precisa de edição.
  O que ele veste e carrega soma POR CIMA disso — quem quiser levar um corpo
  daqui para ali enfrenta o peso da pessoa mais o de tudo que ela leva consigo.
- `skills`: mapa `nome: nível`.
- `body` (opcional, spec 019): o CORPO do personagem, mapa `slot → capacidade`
  (inteiro ≥ 0) — quantas peças cada parte comporta. Ausente ⇒ **corpo humano
  padrão** (`cabeca:1, rosto:1, pescoco:1, torso:1, costas:1, bracos:1, mao:2,
  dedo:10, cintura:1, pernas:1, pes:1`), então nenhuma ficha já escrita precisa de
  edição. DECLARADO, é o corpo COMPLETO daquele ser: os slots que você NÃO listar
  são ausentes (capacidade 0), não herdados do humano — é assim que se faz um
  centauro sem `pes` ou um cão sem `mao`. O vocabulário de nomes de slot é ABERTO:
  invente `asas`, `cauda`, `pata`, `tentaculo` — o item vestível casa pelo mesmo
  nome em `wearable.slot`. Vestir uma peça num slot que o corpo não tem é recusado
  in-world (não serve).
  - **Slot de PEGA** (onde os itens PEGOS/recebidos vão parar — a mão do humano, a
    boca do cão): o valor de um slot pode ser a forma rica `{capacidade: N, pega:
    true}` em vez do inteiro cru. `pega: true` marca aquele slot como o de pega.
    Sem nenhuma marca, o corpo pega com a **mão** (`mao`) se a tiver — por isso
    humano e qualquer corpo que ainda tenha mãos não precisam declarar nada. Um
    corpo sem mãos e sem `pega` não segura nada (só veste).
  - Exemplo de um cão que segura na boca: `body: {cabeca: 1, pescoco: 1, torso: 1,
    cauda: 1, pata: 4, focinho: {capacidade: 1, pega: true}}`.
- `status`: mapa livre; convenção: `hp`, `hp_max`, `hunger`, `fatigue`, `action`
  (ação idle), `mood`, `conditions` (lista).
  - Combate (spec 008) dá uso mecânico a três delas: `hp`/`hp_max` são a
    vitalidade (`hp` nunca desce de 0; sem `hp_max` registrado, o Motor preenche
    `max(1, 10 + mod(CON))` no primeiro golpe que toca o personagem), e
    `conditions` recebe `incapacitado` (hp 0) ou `morto` (golpe deliberado em quem
    já estava caído). Quem tem qualquer uma das duas não age e não defende — e
    nada é apagado: a pasta do caído permanece inteira, com pertences saqueáveis.
Pasta do personagem pode conter `/memories/` e sub-pastas de `item`.

Persuasão (spec 007) NÃO adiciona schema: a vontade do alvo (0–10) é decidida pelo
Árbitro a cada tentativa, pela régua canônica, e nunca é gravada em arquivo. O
persuadido viaja pela rota como qualquer viajante (`transit` normal). O CHA de quem
persuade entra no teste (1–9); personalidade no body, memórias e `status` são a
matéria-prima da nota — escreva-os bem e a persuasão "funciona" sozinha.

## route — `route.md`  (fase 2)
Obrigatórios: `type`, `id`, `name`, `from`, `to`, `travel_time_base`,
`bidirectional`, `prerequisites`.
- `travel_time_base`: segundos de relógio real (inteiro). O tempo real de viagem é
  `travel_time_base + modificadores` (ex.: fadiga alta atrasa). A chegada é resolvida
  de forma preguiçosa na próxima consulta ao mundo (nunca em dois lugares — SC-004).
- `bidirectional`: booleano. Se `true`, a rota também parte de `to` rumo a `from`.
- `prerequisites`: lista de mapas, avaliada em ordem — estáticos primeiro, contextuais
  por último; a 1ª negação interrompe e devolve o motivo (FR-019). Cada mapa tem `id`,
  `type` e um `deny_reason` opcional (mensagem in-world). Tipos suportados no MVP:
  - `none` (estático): sempre passa (placeholder).
  - `item` (estático): exige `required: <item-id>` no inventário do personagem.
  - `attribute` (estático): exige `attribute: <STR|DEX|…>` com `min: <int>`.
  - `status` (contextual): exige `field: <campo de status>` igual a `equals: <valor>`.

## memory — `memory.md`  (fase 3)
Memória de 1ª classe: fica sob `memories/` na pasta do personagem afetado, redigida na
perspectiva dele (o body é a lembrança). Substitui qualquer fila de eventos (FR-026).
Obrigatórios: `type`, `id`, `timestamp_start`, `timestamp_end`, `intensity`, `state`.
- `timestamp_start` / `timestamp_end`: epoch em segundos (relógio real). O Árbitro decide
  o prazo via `ttl_seconds`; o Motor grava `timestamp_end = agora + ttl`.
- `intensity`: `trivial | small | medium | large | giant` — decidida pelo Árbitro; ordena o
  contexto. `trivial` (6h) é o degrau abaixo de `small` (2 dias) — ação mecânica sem
  valência nem progresso (abrir um recipiente, vestir algo), não um episódio vivido.
- `summary` (opcional): rótulo curto (3-6 palavras) exibido na lista de memórias da lateral;
  ao clicar, mostra o body. Se ausente, o server deriva um resumo do começo do body.
- `state`: `active | expired`. A expiração é preguiçosa (na consulta ao mundo): passado o
  prazo, o `state` vira `expired` e a memória some do contexto — o arquivo nunca é deletado.
- Saliência (derivada na leitura, não armazenada): o contexto marca cada memória ativa como
  `vivida` (recente e/ou forte — janela escalada pela intensidade) ou `latente` (antiga). A
  Mente pesa vívidas sempre; latentes só quando a cena as evoca — memória análoga à humana.
- `kind` (opcional, spec 013): `acontecimento | rota`. **Ausente ⇒ `acontecimento`**, e é o
  que mantém válido todo o mundo anterior à spec. `acontecimento` é a lembrança narrativa
  de sempre e ACUMULA (dez negociações = dez memórias); `rota` é conhecimento de caminho,
  é ÚNICA por rota e RENOVA o prazo a cada uso.
- `involved` (opcional, spec 013): lista de ids dos personagens que participaram, sem o
  dono da lembrança. **Ausente ⇒ lista vazia.** É o que permite ao mundo responder "ele
  conhece esta pessoa?" — antes disso, quem mais esteve na cena existia só na prosa.
- `about` (opcional, spec 013): id da entidade lembrada. Obrigatório em `kind: rota` (é a
  rota que ele sabe percorrer), ausente em `acontecimento`.
- Memórias de `kind: rota` NÃO descem ao client: são maquinaria de viagem, e a Mente não
  precisa delas para narrar. O mundo as consulta direto.

## intention — `intention.md`  (spec 026)
Intenção persistente: fica sob `intentions/` na pasta do personagem, redigida na
perspectiva dele (o body é o compromisso/plano, em linguagem livre — motivação,
progresso, condição de conclusão/abandono, o que fizer sentido). Não é quest nem
tarefa do servidor: é só conhecimento que participa da decisão da LLM, do mesmo
jeito que memória — a diferença é que UMA intenção pode ser reescrita no lugar
(o mesmo arquivo, o mesmo id), porque é um plano vivo, não um evento acumulado.
Obrigatórios: `type`, `id`, `status`, `created_ts`, `updated_ts`.
- `status`: `ativa | concluida | abandonada`. Só este campo é estruturado — o
  Motor lê/transiciona o estado, nunca interpreta o conteúdo (Princípio XI).
  Não há decaimento por relógio (ao contrário de memória): encerrar é sempre
  decisão explícita de quem a escreve.
- `created_ts` / `updated_ts`: epoch em segundos (relógio real), só para rastro —
  nenhuma lógica do Motor depende deles além de gravá-los.
- Ao encerrar (`concluida`/`abandonada`), o arquivo NUNCA é apagado (Princípio
  IV) — só sai da lista de intenções ativas que desce ao contexto.
- Uma intenção encerrada não pode ser reaberta por `set_intention`: uma
  continuação nasce como intenção NOVA, com o próprio id.

## item — `item.md`
Obrigatórios: `type`, `id`, `name`.
Material que ENSINA (mapa, carta, inscrição) é um item **comum**: não há campo para
isso. Quem cria escreve, no corpo, os caminhos que a peça mostra — o mundo lê a prosa
e deriva o aprendizado (spec 014). Um item que ensina e um que não ensina têm
exatamente os mesmos campos; a diferença está só no texto.
Itens podem conter outros itens (aninhamento por pastas), e podem estar dentro de
um `object` (ex.: loot de um baú, ver abaixo) — nesse caso ficam ocultos do
contexto até uma ação os liberar. Opcional: `interactions` (ver seção `object`
abaixo — mesma forma e regras). Futuramente: `travel_modifier` para modificar
tempo de rota.

### Física do item (spec 004 — equipamentos e partes do corpo)

Campos opcionais no topo (itens antigos continuam válidos):

- `size`: tamanho na régua **`PP < P < M < G < XG < XXG < XXXG`**. Ausente ⇒ `P`.
  Critério físico: `PP` cabe fechado numa mão (moeda, anel); `P` segura-se com uma
  mão (frasco, adaga); `M` carrega-se com uma mão/braço (mochila, atiçador); `G`
  exige o corpo/duas mãos (espada longa, baú pequeno); `XG` não se carrega sozinho
  (geladeira, arca); `XXG` escala de construção (casa, carroça); `XXXG` escala de
  terreno (montanha).
- `weight_kg`: peso em kg (número > 0). Ausente ⇒ padrão da classe de `size`:
  `PP 0.1 · P 1 · M 5 · G 25 · XG 100 · XXG 5000 · XXXG 1000000000`. O peso
  efetivo de um contêiner soma o conteúdo, recursivamente.
- `wearable`: presença torna o item **vestível**. Mapa com `slot` OBRIGATÓRIO —
  uma parte canônica do corpo (tabela abaixo). Qualquer item pode ser *segurado*
  numa mão (segurar ≠ vestir); `wearable` habilita as demais partes.
  - `speed_multiplier` (opcional, spec 009): número > 0 que ACELERA a viagem por
    rota enquanto a peça está VESTIDA — nunca guardada, nunca na mão. Vale apenas
    o MAIOR entre os itens vestidos: multiplicadores não se somam nem se
    multiplicam (bota 2 + capa 3 = 3, não 6). O tempo de viagem é dividido por
    ele como última etapa, depois dos demais modificadores. Ausente ⇒ sem efeito;
    valor ≤ 1 é aceito mas inerte (equipamento não atrasa ninguém). Mora dentro
    de `wearable` porque só faz sentido vestido.
- `container`: presença torna o item **contêiner**. Mapa com AMBOS obrigatórios:
  `max_size` (maior tamanho aceito; NUNCA maior que o `size` do próprio item —
  nada entra maior que o recipiente) e `max_items` (inteiro ≥ 1; conta apenas os
  itens diretamente dentro — uma bolsa cheia ocupa 1 vaga da mochila).
- `state.slot`: gerido pelo Motor, NÃO editar à mão em jogo. Invariante: item
  filho direto de um personagem ⟺ `state.slot` presente (`mao` = segurado; outro
  slot = vestido). Item aninhado em contêiner não tem `state.slot` (está
  guardado, oculto do contexto dos outros).

### Preço e disponibilidade (spec 011 — comércio)

Quatro campos opcionais no topo. **A ausência significa INDISPONÍVEL** para aquele
negócio — um mundo sem essas marcas simplesmente não comercia, e isso é
intencional: comércio existe onde o autor escreve que existe.

```yaml
value: 12          # quanto a coisa vale (número >= 0)
for_sale: true     # o dono aceita VENDER isto por dinheiro
negotiable: true   # o dono aceita TROCAR isto por bens
currency: true     # isto É dinheiro
```

- **Comprar** exige, na mercadoria, `for_sale: true` E `value`; e no pagamento,
  `currency: true` E `value`. Faltando qualquer um, o item não é vendável — o
  Árbitro NÃO estima preço.
- **Trocar** exige `negotiable: true` em TODO item que sai, dos dois lados —
  inclusive de quem propõe. O relicário de família não entra em negócio porque o
  mundo não o põe na mesa; nenhuma avaliação de cena contorna isso.
- As marcas são independentes: dá para estar à venda e não ser trocável, e
  vice-versa.
- Mudar essas marcas em jogo (passar a aceitar vender algo) é capacidade futura;
  hoje elas são dado editorial.

### Arma e armadura (spec 008 — combate)

Dois blocos opcionais no topo; itens antigos continuam válidos.

```yaml
weapon:
  damage: 6           # inteiro >= 1 — dano-base do golpe
  attribute: STR      # STR (corpo a corpo) ou DEX (leve/arremesso)
armor:
  protection: 2       # inteiro >= 0 — absorção quando VESTIDA
```

- `weapon`: ambos os campos obrigatórios dentro do bloco (declaração parcial é
  inválida). Segurar na mão continua sendo o requisito físico; `weapon` só diz o
  que o golpe vale. Item SEM `weapon` usado para atacar vale como **improvisado**
  (`damage 1`, `attribute STR`) — assim como o golpe desarmado.
- `armor`: exige `wearable` no mesmo item (o que não se veste não protege) e só
  conta quando efetivamente VESTIDO — guardado ou segurado protege nada. A
  absorção de um personagem é a soma das peças vestidas.

Partes canônicas do corpo (multiplicidade entre parênteses): `cabeca` (1),
`rosto` (1), `pescoco` (1), `torso` (1), `costas` (1), `bracos` (1), `mao` (2),
`dedo` (10), `cintura` (1), `pernas` (1), `pes` (1). Pares vestidos como conjunto
(calça, botas) são slot único; mãos e dedos contam por unidade.

Capacidades do personagem (derivadas de STR, nunca armazenadas):
**carregar = STR × 7 kg** (teto do peso total consigo) e **empurrar = STR × 14 kg**
(teto para deslocar sem carregar). Não existe inventário solto: um item só está
com um personagem vestido, segurado ou guardado num contêiner que ele carrega.

### Fecho e travas de contêiner (spec 005)

Vale para item-contêiner E para `object` que guarda itens (baú):

- `state.fechado` (opcional, bool): `true` = fechado. Ausente ⇒ ABERTO (todo o
  mundo antigo continua aberto). Mutável SÓ pelas ações de abrir/fechar
  arbitradas — nunca por mutação direta.
  - **`false` DECLARADO é diferente de ausente** (spec 011): significa
    escancarado de propósito, e aí o conteúdo fica à mostra para os OUTROS —
    a caixa do mascate aberta na praça, que não precisa ser perguntada.
    Ausente é o padrão de qualquer bolsa: privado, mas utilizável. Sem essa
    distinção, o dinheiro que cada um carrega no bolso viraria público.
- **Fechado esconde o conteúdo de todos** (nem o portador vê): nada dentro
  aparece em contexto/inventário/observação, e não se guarda nem tira até abrir.
  A física NÃO muda: o conteúdo continua pesando e ocupando vagas.
- `locks` (opcional, topo do frontmatter): travas de abertura e/ou fechamento.
  Em `item`, exige o bloco `container` no mesmo item; em `object` é livre.

```yaml
locks:
  open:                    # TODAS as camadas precisam passar (conjunção)
    - {type: item, required: chave-de-ferro}
    - {type: item, required: chave-de-prata, deny_reason: "a fechadura prateada resiste"}
  close:                   # lista independente; ausente = fecha livremente
    - {type: item, required: chave-de-ferro}
```

- Tipo de trava do MVP: `item` — quem age precisa ter o item do id `required`
  consigo, alcançável por contêineres ABERTOS e **fora do contêiner alvo** (a
  chave dentro do próprio baú nunca o abre nem fecha — por isso um baú cujo
  `close` exige a chave jamais a tranca dentro). A chave não é consumida.
  `deny_reason` (opcional) é a recusa in-world preferida na narração.
- Aviso autoral: mundo escrito com a chave DENTRO do contêiner fechado que ela
  abre gera aviso (não erro) no boot e em `/api/world/health` — sem arrombamento
  no MVP, isso é um deadlock.

## object — `object.md`  (fase 5)
Entidade física não portável, ancorada numa `location` ou numa `route` — nunca
dentro de um `character` (ela não vai para inventário).
Obrigatórios: `type`, `id`, `name`.
Opcionais:
- `state`: mapa livre, mutável pelo Árbitro (mesmo espírito do `status` de
  `character`, mas sem atributos/skills). Mudanças físicas permanentes (ex.: uma
  fechadura arrombada) persistem aqui, direto no arquivo — nunca dependem de uma
  memória ativa para "lembrar" o estado.
- `interactions`: lista de ações sugeridas, cada uma com `action` (obrigatório
  dentro da entrada), `requires` (ex.: `{skill: nome, min_level: N}` ou
  `{item: id-do-item}`), `check` (ex.: `{attribute: DEX, dc: 14}`) e `hint`
  (texto livre). **Consultivo**: o Árbitro usa como contexto para decidir com
  mais consistência, mas nunca é obrigado a seguir o `check` literalmente — a
  resolução final é sempre julgamento dele. Nenhuma `interaction` gera tool nem
  código novo; a ação `inspect` está sempre disponível mesmo sem `interactions`
  declaradas.

Identidade do object (`id`, `name`, `type`, `interactions`) é imutável por ação —
só o bloco `state` é mutável. Um `item` aninhado dentro de um `object` pode mudar
de posse para o inventário de um personagem presente (`pegar`/`saquear`), assim
como um `item` pode ser transferido entre dois personagens presentes (`dar`) —
capacidade do Árbitro, não um campo de frontmatter.
