# LibreControl

Um framework open-source de Command & Control projetado para a sala de aula, não para o campo de batalha — desmistificando a orquestração de adversários, um pacote por vez.

> **Aviso**: O LibreControl é estritamente para uso educacional e pesquisa de segurança autorizada. Os autores não são responsáveis por uso indevido. Não utilize este software em redes que você não possui ou para as quais não tenha permissão explícita de teste.

## A Filosofia

Segurança muitas vezes é ensinada de forma abstrata, mas adversários operam de forma concreta. O **LibreControl** existe para quebrar essa opacidade.

> Não estamos construindo uma arma; estamos construindo uma janela.

Frameworks C2 existentes são projetados para furtividade e velocidade. Este projeto foi projetado para **legibilidade e compreensão**. Nós removemos a obfuscação para revelar a mecânica bruta da orquestração, evasão e persistência.

> Para entender a sombra, você deve primeiro entender a luz que a projeta.

## Como Funciona

O LibreControl disseca a anatomia de um ataque em seus componentes atômicos. Ele fornece um laboratório seguro e isolado para observar como agentes se comunicam, como comandos são organizados e como a persistência é mantida.

### Pilares Centrais

* **O Sistema Nervoso (Canais)**: Explore como os dados se movem. Implementamos protocolos de comunicação modulares (HTTP/S, DNS, SMB, etc.) para demonstrar como o tráfego se mistura ao ruído normal da rede.
* **A Camuflagem (Evasão)**: Um estudo sobre “Evasão de Defesa”. Veja como assinaturas são mascaradas e como análises heurísticas são contornadas.
* **A Âncora (Persistência)**: Mecanismos que garantem que o agente sobreviva a reinicializações e tentativas de correção, demonstrando a resiliência das ameaças modernas.

### Arquitetura

* **O Cérebro (Servidor)**: Responsável por gerenciar estado e agendar tarefas.
* **As Mãos (Agente)**: O payload executado no alvo, responsável por realizar as instruções.
* **O Listener**: A interface que conecta os dois, abstraindo o protocolo de comunicação.

## Junte-se à Causa

Este projeto pertence aos estudantes, pesquisadores e curiosos.

* **Leia o Código**: Priorizamos comentários claros em vez de hacks engenhosos.
* **Quebre o Código**: Encontre falhas na nossa lógica de evasão e corrija-as.
* **Contribua**: Envie um PR com um novo canal de comunicação ou um método de persistência.
