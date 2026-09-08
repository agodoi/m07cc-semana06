# Computação Ágil e Elástica (parte prática: como medir)

Neste encontro vamos montar, na prática, uma arquitetura que aguenta crescimento de acessos: um balanceador de carga (ELB) na frente de um grupo de servidores que cresce e encolhe sozinho (Auto Scaling). Depois, vamos **medir** esse comportamento: gerar carga, observar os alarmes do CloudWatch e ver o grupo escalar para cima e para baixo.

> A parte conceitual do encontro (6 R's da migração, Well-Architected Framework, mitigação de DDoS) está nos slides. Aqui o foco é mão na massa.

## Objetivos

Ao final desta instrução você será capaz de:

1. Criar uma **AMI** a partir de uma instância configurada e explicar para que ela serve.
2. Criar um **Application Load Balancer** com um **grupo de destino** e um **health check**.
3. Criar um **modelo de execução** e um **grupo do Auto Scaling** ligado ao ELB.
4. Verificar que o balanceamento funciona acessando a aplicação pelo DNS do ELB.
5. Gerar carga, observar os alarmes do CloudWatch e ver o grupo **escalar para cima (scale-out)**.
6. Encerrar a instância **Web Server 1** após confirmar o funcionamento do Auto Scaling, seguindo a sequência oficial do Laboratório 6.
7. *(Atividade complementar)* Observar o grupo **escalar para baixo (scale-in)** quando a carga cessa.
8. *(Caminho B)* Usar o **K6** a partir de um bastion host para gerar carga controlada contra o ELB.

## Pré-requisitos

- Learner Lab iniciado, com o par de chaves **vockey** disponível.
- **Caminho A:** ter concluído o laboratório do Módulo 10 do AWS Foundation até o ponto em que a instância **Web Server 1** existe e está em execução (é a mesma da instrução [EC2-RDS](https://github.com/agodoi/EC2-RDS)).
- **Caminho B:** ter concluído os PASSOS 01 a 08 da instrução [ArquiteturaCorp](https://github.com/agodoi/ArquiteturaCorp/blob/main/README.md).

## Tempo estimado

| Bloco Tempo                             |          |
| --------------------------------------- | -------- |
| Passos 1 a 3 (AMI, ELB, Auto Scaling)   | 40 min   |
| Passo 4 (verificar balanceamento)       | 10 min   |
| Passo 5 (teste de carga e observação)   | 25 min   |
| Passo 6 (scale-in) e Passo 7 (encerrar) | 15 min   |
| Caminho B completo (com K6)             | + 45 min. Quem fizer e me enviar um vídeo (No slack) até 23h59 de hoje, será gratificado. E essa parte é um entregável na próxima sprint. |

## Impactos no seu projeto

A aplicação da Globoplay precisa de auto scaling e de balanceamento de carga entre servidores internos. Esta instrução ensina exatamente isso. O **Caminho B** é o mais próximo do que você vai usar no projeto.

## Não faça

- **Nunca** tenha 20 ou mais instâncias em execução simultânea (de qualquer tamanho). Isso causa a desativação imediata da conta AWS e a exclusão de todos os recursos.
- **Não** rode testes K6 do seu computador do Inteli contra a AWS. O teste sai do bastion host, dentro da VPC.
- **Não** teste outros sites (incluindo o do Inteli). Isso é tratado como ataque e você será banido.

## Conceitos que você vai ver na prática

Antes de clicar, alinhe o vocabulário. Você vai ser cobrado nas perguntas de reflexão.

- **Escalabilidade** é a capacidade de aumentar recursos para atender mais demanda (na horizontal, mais máquinas; na vertical, máquinas maiores).
- **Elasticidade** é escalar **automaticamente** nos dois sentidos: crescer no pico e encolher na baixa, pagando só pelo que usa.
- **Alta disponibilidade** é continuar respondendo mesmo quando um componente falha. Aqui isso vem de distribuir instâncias em **duas Zonas de Disponibilidade** e de o health check do ELB evitar o envio de tráfego para destinos que não respondem.
- **Elastic Load Balancer (ELB)** distribui o tráfego de entrada entre várias instâncias EC2. Nesta arquitetura, ele é a porta de entrada da aplicação e encaminha tráfego apenas para destinos considerados íntegros. [Definição](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-ElasticLoadBalancer.md)
- **Grupo de destino** é o recurso associado ao ELB que mantém os destinos registrados e executa o health check neles. [Definição](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-GrupoDeDestino.md)
- **AMI (Amazon Machine Image)** é uma imagem usada como modelo para iniciar novas instâncias, contendo a configuração necessária para reproduzir o servidor. O Auto Scaling usa a AMI definida no modelo de execução para criar novas instâncias sem que você configure tudo de novo.
- **Auto Scaling** mantém o número de instâncias entre um mínimo e um máximo e ajusta esse número conforme uma métrica (aqui, uso médio de CPU no Caminho A e requisições do ALB por destino no Caminho B).

Tipos de balanceador que você vai ver no console:

| Tipo Camada         |          OSI      | Uso principal                                                                                                              |
| ----------------------------------- | -------------- | ------------------------------------------------------------------------------------------------------------- |
| **Application Load Balancer (ALB)** | 7 (aplicação)  | HTTP/HTTPS, roteia por URL, cabeçalhos etc. Ideal para aplicações web e microsserviços. **É o que usaremos.** |
| **Network Load Balancer (NLB)**     | 4 (transporte) | TCP/UDP de altíssimo desempenho e baixa latência (jogos, streaming).                                          |
| **Gateway Load Balancer (GLB)**     | 3 (rede)       | Insere appliances virtuais (firewall, IDS/IPS) no caminho do tráfego.                                         |

### Arquitetura inicial

 <img src="https://github.com/agodoi/TesteCargaAWS/blob/main/imgs/starting-architecture.png" width="500"> 

### Arquitetura final

 <img src="https://github.com/agodoi/TesteCargaAWS/blob/main/imgs/final-architecture.png" width="500"> 

### Sobre o ícone de atualizar

Várias telas do console demoram alguns segundos para refletir o que acabou de acontecer. Quando a instrução disser **"atualize a tela"**, clique no ícone circular de atualizar no canto superior direito da lista (a "rodinha"). Não use o F5 do navegador, que às vezes te joga para outra página.

---

## Escolha o seu caminho

| **Caminho A: Laboratório do Módulo 10** **Caminho B: Arquitetura Corporativa**  |                                         |                                                  |
| --------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------------ |
| Quando usar                                                                       | **Na aula de hoje.**                    | No seu projeto, ou se sobrar tempo.              |
| Ponto de partida                                                                  | Instância **Web Server 1** do Módulo 10 | VPC\_Arquitetura\_Corp com bastion + EC2 privado |
| Como gerar carga                                                                  | Botão **Load Test** da aplicação PHP    | **K6** rodando no bastion host                   |
| Por onde começar                                                                  | **Passo-01**                            | Seção **Caminho B**, no final deste documento    |

Os Passos 01 a 07 abaixo descrevem o **Caminho A**. O Caminho B reaproveita a lógica deles com os nomes da sua rede e é descrito separadamente para não misturar as coisas.

---

# CAMINHO A: Laboratório do Módulo 10

## Passo-01: Criar uma AMI para o Auto Scaling

Você vai tirar uma "foto" da instância **Web Server 1**. É dessa foto que o Auto Scaling vai criar as cópias.

**1.1)** No Console de Gerenciamento da AWS, na caixa de pesquisa ao lado de Serviços, pesquise e escolha **EC2**.

**1.2)** No painel à esquerda, selecione **Instâncias**. Confirme que **Web Server 1** está em execução e que as **Verificações de status** mostram **2/2 verificações aprovadas** em verde. Se necessário, atualize a tela.

**1.3)** Selecione **Web Server 1**.

**1.4)** No menu **Ações**, selecione **Imagem e modelos** > **Criar imagem** e configure:

- Nome da imagem: **WebServerAMI**
- Descrição da imagem: **Lab AMI for Web Server**
- Deixe o restante como está.

**1.5)** Clique em **Criar imagem**. Um banner verde exibe o ID da nova AMI (algo como `ami-065d3b82f7a510b8e`).

**Checkpoint:** no painel à esquerda, em **Imagens > AMIs**, a **WebServerAMI** aparece com status **Disponível** (pode levar alguns minutos). Só siga quando estiver disponível.

> **Reflexão 1:** se você mudar o código da aplicação amanhã, o que precisa acontecer para que as novas instâncias do Auto Scaling recebam a mudança?

## Passo-02: Criar o grupo de destino e o ELB

### 2.A: Grupo de destino

**2.1)** No painel à esquerda, em **Balanceamento de carga**, escolha **Grupos de destino**.

**2.2)** Escolha **Criar grupo de destino**.

**2.3)** Tipo de destino: **Instâncias**.

**2.4)** Nome do grupo de destino: **LabGroup**.

**2.5)** VPC: selecione **Lab VPC**.

**2.6)** Observe a seção **Verificações de integridade**. O grupo de destino vai fazer um GET no caminho `/` de cada instância. Mantendo a configuração padrão do Application Load Balancer, o código de sucesso esperado é **HTTP 200**. É possível configurar outros códigos ou intervalos, mas nesta prática não vamos alterar esse valor. Lembrando as classes de código HTTP:

- 100–199: informativas
- 200–299: sucesso
- 300–399: redirecionamento
- 400–499: erro do cliente
- 500–599: erro do servidor

Não altere nada e selecione **Próximo**.

**2.7)** Na tela **Registrar destinos**, **não marque nada**. As instâncias serão registradas automaticamente pelo Auto Scaling no Passo-03. Se aparecer o Web Server 1 ou qualquer outra instância, ignore.

**2.8)** Selecione **Criar grupo de destino**.

**Checkpoint:** o **LabGroup** aparece na lista com **0 destinos**. É isso mesmo.

### 2.B: Balanceador de carga

**2.9)** No painel à esquerda, escolha **Balanceadores de carga** e depois **Criar balanceador de carga**.

**2.10)** Em **Application Load Balancer**, selecione **Criar**.

**2.11)** Nome do balanceador de carga: **LabELB**.

**2.12)** Role até **Mapeamento de rede**. Em **VPC**, selecione **Lab VPC**. [Definição](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-MapeamentoRede.md)

**2.13)** O ELB vai receber tráfego da internet, então ele precisa estar em **sub-redes públicas**, uma em cada Zona de Disponibilidade:

- Marque a **primeira** Zona de Disponibilidade e selecione **Sub-rede pública 1**.
- Marque a **segunda** Zona de Disponibilidade e selecione **Sub-rede pública 2**.

> **Reflexão 2:** por que o ELB precisa de duas Zonas de Disponibilidade? O que acontece com a aplicação se uma zona inteira cair?

**2.14)** Em **Grupos de segurança**, selecione **Web Security Group** e remova o grupo **default** clicando no X. Só o Web Security Group deve permanecer. [Definição](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-WebSecurityGroup.md)

**2.15)** Em **Listeners e roteamento**, na linha **HTTP:80**, defina a ação padrão para encaminhar para **LabGroup**.

> Se o menu estiver vazio, o grupo de destino não foi criado na VPC certa. Volte ao item 2.5.

**2.16)** Deixe o restante como está e selecione **Criar balanceador de carga**.

**Checkpoint:** o **LabELB** aparece com estado **Provisionando** e, após alguns minutos, **Ativo**. Você pode continuar para o Passo-03 enquanto ele provisiona.

## Passo-03: Criar o modelo de execução e o grupo do Auto Scaling

### 3.A: Modelo de execução

O modelo de execução diz ao Auto Scaling **como** criar cada instância: qual AMI, qual tamanho, qual chave, qual grupo de segurança. [Definição](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-ModeloDeExecucao.md)

**3.1)** No painel à esquerda, em **Instâncias**, selecione **Modelos de execução** e depois **Criar modelo de execução**.

**3.2)** Nome do modelo de execução: **LabConfig**.

**3.3)** Em **Orientação sobre o Auto Scaling**, marque **Fornecer orientação para me ajudar a configurar um modelo que eu possa usar com o EC2 Auto Scaling**.

**3.4)** Em **Imagens da aplicação e do SO**, clique em **Minhas AMIs** e selecione **WebServerAMI**.

**3.5)** Tipo de instância: **t2.micro**.

**3.6)** Nome do par de chaves: selecione o par disponibilizado pelo Learner Lab: normalmente **vockey**.

**3.7)** Em **Configurações de rede**, **não** selecione sub-rede (o grupo do Auto Scaling decide isso). Em **Firewall (grupos de segurança)**, marque **Selecionar grupo de segurança existente** e escolha **Web Security Group**.

**3.8)** Expanda **Detalhes avançados**, role até **Monitoramento detalhado do CloudWatch** e selecione **Habilitar**. Isso faz as métricas chegarem a cada 1 minuto em vez de 5, o que deixa o Auto Scaling reagir mais rápido.

**3.9)** Deixe o restante como está e clique em **Criar modelo de execução**.

**3.10)** Na tela de sucesso, clique no link do modelo **LabConfig**.

### 3.B: Grupo do Auto Scaling

**3.11)** No menu **Ações**, selecione **Criar grupo do Auto Scaling**. O assistente tem 7 etapas; acompanhe no painel à esquerda.

**Etapa 1: Modelo de execução**

- Nome do grupo: **Lab Auto Scaling Group**
- Modelo de execução: confirme que é **LabConfig**
- Selecione **Próximo**

**Etapa 2: Opções de execução da instância**

- VPC: **Lab VPC**
- Zonas de disponibilidade e sub-redes: **Sub-rede privada 1** e **Sub-rede privada 2**

> Repare na diferença: o **ELB** ficou nas sub-redes **públicas** (ele recebe tráfego da internet); as **instâncias** ficam nas **privadas** e não são diretamente acessíveis pela internet. Nesta arquitetura do laboratório, o tráfego da aplicação chega às instâncias por meio do ELB.

- Selecione **Próximo**

**Etapa 3: Opções avançadas**

- Marque **Anexar a um balanceador de carga existente**.
- Em **Grupos de destino de balanceador de carga existentes**, selecione **LabGroup | HTTP**.
- Em **Configurações adicionais**, marque **Habilitar coleta de métricas de grupo no CloudWatch**.
- Selecione **Próximo**

**Etapa 4: Tamanho do grupo e políticas de escalabilidade**

- Capacidade desejada: **2**
- Capacidade mínima: **2**
- Capacidade máxima: **6**

> Com esses valores o grupo nunca terá menos de 2 nem mais de 6 instâncias. Mesmo assim, **lembre-se da regra dos 20 EC2**. Se você tiver outros grupos ou instâncias rodando, some tudo.

- Em **Escalabilidade**, marque **Política de escalabilidade de rastreamento de destino** (o nome pode aparecer como "dimensionamento com monitoramento do objetivo") e configure: 
  - Nome da política: **LabScalingPolicy**
  - Tipo de métrica: **Média de utilização da CPU**
  - Valor de destino: **60**

> Isso instrui o Auto Scaling a manter a CPU média do grupo perto de 60%. Se subir, ele adiciona instâncias; se cair, ele remove.

- Selecione **Próximo**

**Etapa 5: Notificações**

- Não altere nada. Selecione **Próximo**. (No projeto, vale configurar um e-mail via SNS para saber quando o grupo escalar.)

**Etapa 6: Tags**

- Clique em **Adicionar tag**: Chave **Name**, Valor **Lab Instance**.
- Selecione **Próximo**

**Etapa 7: Revisão**

- Confira e clique em **Criar grupo do Auto Scaling**.

**Checkpoint:** o grupo aparece com **0 instâncias** e, em 1 a 2 minutos, com **2**. Atualize a tela até ver 2.

> **Aprofundamento:** em uma arquitetura de produção, também é possível configurar o Auto Scaling Group para considerar os health checks do Elastic Load Balancing ao avaliar a integridade das instâncias. **Neste Caminho A, não altere essa configuração**, pois o objetivo é manter o ambiente exatamente alinhado às etapas do Laboratório 6 oficial da AWS.

## Passo-04: Verificar se o balanceamento está funcionando

**4.1)** No painel à esquerda, selecione **Instâncias**. Devem existir duas novas instâncias chamadas **Lab Instance**. Se não aparecerem, aguarde 30 segundos e atualize a tela.

**4.2)** No painel à esquerda, escolha **Grupos de destino**, clique em **LabGroup** e abra a aba **Destinos**.

**4.3)** Duas instâncias **Lab Instance** devem estar listadas. Aguarde até que o **Status** das duas mude para **íntegro** (healthy). Atualize a tela se necessário.

> **Íntegro** significa que a instância respondeu ao health check com o código de sucesso configurado: nesta prática, **HTTP 200**: e que o ELB pode encaminhar tráfego para ela. Uma instância **não íntegra** deixa de receber tráfego do balanceador enquanto permanecer nesse estado.

**4.4)** No painel à esquerda, escolha **Balanceadores de carga** e clique em **LabELB**.

**4.5)** No painel **Detalhes**, copie o **Nome do DNS** (algo como `LabELB-1152052616.us-east-1.elb.amazonaws.com`, sem o "(Registro A)"). Não confunda com o ARN.

**4.6)** Abra uma nova aba do navegador, cole o nome do DNS e pressione Enter. A aplicação da agenda (a mesma da instrução EC2-RDS) deve aparecer.

**Checkpoint / Parabéns:** a requisição entrou pelo ELB, foi encaminhada para uma das duas instâncias privadas e a resposta voltou até você. Recarregue algumas vezes: as requisições são distribuídas pelo balanceador entre os destinos íntegros. Não é necessário que cada atualização do navegador alterne obrigatoriamente entre as duas instâncias.

> **Reflexão 3:** se você encerrar uma das duas Lab Instance à mão, o que você espera que aconteça (a) com a aplicação e (b) com o número de instâncias, e em quanto tempo? Se der tempo, teste no final.

## Passo-05: Gerar carga e observar o scale-out

Hoje o grupo tem 2 instâncias porque o mínimo é 2 e não há carga. Agora você vai forçar a CPU a subir e ver o Auto Scaling responder.

### 5.A: Localizar os alarmes

**5.1)** Mantenha a aba da aplicação aberta. Em outra aba, no console, pesquise e selecione **CloudWatch**.

**5.2)** No painel à esquerda, expanda **Alarmes** e selecione **Todos os alarmes**. Dois alarmes devem aparecer. Eles foram criados automaticamente pela política de rastreamento de destino do Auto Scaling. O objetivo é manter a utilização média de CPU próxima a **60%**, respeitando os limites de **2 a 6 instâncias**.

> **Somente se os alarmes não aparecerem em 60 segundos**, mesmo após atualizar a tela, siga o procedimento previsto no Laboratório 6 oficial:
>
> 1. Volte para **EC2**.
> 2. No painel à esquerda, selecione **Grupos do Auto Scaling**.
> 3. Selecione **Lab Auto Scaling Group**.
> 4. Abra a aba **Escalabilidade automática**.
> 5. Selecione **LabScalingPolicy** e escolha **Ações > Editar**.
> 6. Altere o **Valor de destino** para **50** e selecione **Atualizar**.
> 7. Volte ao **CloudWatch > Todos os alarmes** e confirme que os dois alarmes foram criados.
>
> Se você precisou realizar esse ajuste, considere **50%**, e não 60%, como o novo valor de destino durante as observações seguintes.

**5.3)** Selecione o alarme que está em estado **OK** e que possui **AlarmHigh** no nome. O estado **OK** indica que ele ainda não foi acionado. O gráfico deve mostrar utilização de CPU baixa.

> No fluxo padrão do laboratório, o alarme de scale-out está relacionado à utilização média de CPU acima do valor de destino. Se você manteve a configuração original, esse valor é **60%**; se executou o procedimento de contingência acima, ele passou a ser **50%**.

### 5.B: Gerar carga

**5.4)** Volte à aba da aplicação e clique em **Load Test**, ao lado do logotipo da AWS. O aplicativo passa a realizar cálculos para aumentar a utilização da CPU, e a página é atualizada automaticamente para gerar carga nas instâncias do grupo do Auto Scaling. **Não feche esta aba.**

**5.5)** Volte ao CloudWatch. A cada 60 segundos, atualize a tela. Em menos de 5 minutos, você deve observar o **AlarmLow** mudar para **OK** e o **AlarmHigh** mudar para **Em alarme**.

- o gráfico do **AlarmHigh** deve indicar aumento da utilização média de CPU;
- no fluxo padrão do laboratório, quando a CPU permanecer acima da linha de **60% por mais de aproximadamente 3 minutos**, o Auto Scaling deverá adicionar instâncias;
- se você alterou anteriormente o valor de destino para **50%**, interprete o gráfico considerando esse novo valor.

**5.6)** Aguarde até que o **AlarmHigh** esteja em estado **Em alarme**. Em seguida, vá ao **EC2 > Instâncias**. Deve haver **mais de duas** instâncias chamadas **Lab Instance**. Elas foram criadas pelo Auto Scaling em resposta ao alarme do CloudWatch.

**5.7)** Abra **EC2 > Grupos do Auto Scaling > Lab Auto Scaling Group > Atividade**. Observe as ações de lançamento registradas pelo grupo e leia pelo menos uma delas.

**Checkpoint:** você viu, na ordem, CPU subir → alarme disparar → novas instâncias serem iniciadas → novas instâncias entrarem no grupo. Isso é **scale-out**.

> **Reflexão 4:** por que uma política de Auto Scaling não deve reagir a qualquer pico isolado de poucos segundos? O que poderia acontecer com a estabilidade da aplicação e com os custos se o grupo aumentasse e diminuísse a capacidade a cada oscilação momentânea?

## Passo-06: Encerrar a instância Web Server 1

A **Web Server 1** serviu para gerar a AMI utilizada pelo grupo do Auto Scaling. Depois que as novas **Lab Instance** foram criadas a partir dessa AMI, a instância original não é mais necessária para a execução da aplicação. Este passo corresponde à **Tarefa 6 do Laboratório 6 oficial da AWS**.

**6.1)** Em **Instâncias**, selecione **apenas** a **Web Server 1**. Confira duas vezes que nenhuma **Lab Instance** está marcada.

**6.2)** No menu **Estado da instância**, selecione **Encerrar instância**.

**6.3)** Na janela de confirmação, selecione **Encerrar**.

> Isso também ajuda a compreender que a AMI utilizada pelo modelo de execução é independente da instância que lhe deu origem. Se futuramente a aplicação mudar, será necessário gerar uma nova AMI e atualizar o modelo de execução utilizado pelo Auto Scaling.

## Envio do trabalho (Laboratório 6: Módulo 10)

- Selecione **Enviar** no topo das instruções do laboratório e confirme com **Sim**.
- Após alguns minutos, o painel de notas mostra a pontuação por tarefa. Se não aparecer, selecione **Notas**. Você pode enviar várias vezes; o último envio é o que vale.
- Para ver o feedback detalhado, selecione **Relatório de envio**.

> **Importante:** até este ponto, o Caminho A segue a ordem operacional do **Laboratório 6 oficial da AWS**. Faça o envio e confira o resultado da correção antes de executar a atividade complementar abaixo. Se precisar corrigir alguma tarefa e reenviar, faça isso antes de prosseguir.

## Atividade complementar opcional: observar o scale-in

Elasticidade acontece nos dois sentidos. O Laboratório 6 oficial encerra a parte avaliada após o scale-out e o encerramento da Web Server 1, mas, se houver tempo de aula, você pode observar também a redução automática da capacidade.

**C.1)** Feche a aba da aplicação que está executando o **Load Test**. Isso interrompe a geração de carga.

**C.2)** No CloudWatch, acompanhe o **AlarmHigh** voltar para **OK** e observe o comportamento do alarme associado ao scale-in. O Target Tracking costuma ser mais conservador ao remover capacidade, portanto a redução pode levar mais tempo do que o scale-out.

**C.3)** Em **EC2 > Grupos do Auto Scaling > Lab Auto Scaling Group > Atividade**, observe se instâncias excedentes são encerradas até o grupo retornar à capacidade mínima de **2**.

> Esta etapa é **complementar** e não faz parte das tarefas avaliadas descritas no Laboratório 6 oficial. Não altere manualmente a capacidade desejada apenas para forçar o resultado.

> **Reflexão 5:** por que o scale-in costuma ser mais conservador que o scale-out? Pense no impacto de remover capacidade cedo demais em comparação com manter capacidade excedente por alguns minutos.

## Finalizando o laboratório

- Selecione **Finalizar Laboratório** no topo da página e confirme com **Sim**.
- Aguarde a mensagem indicando que a exclusão foi iniciada e feche o painel pelo **X**.

---

# CAMINHO B: Arquitetura Corporativa com teste K6

> **Atenção:** a partir daqui você saiu do **Laboratório 6 oficial da AWS**. As etapas B-1 a B-9 são uma aplicação dos mesmos conceitos na arquitetura do projeto e **não fazem parte das tarefas avaliadas do Lab 6**.

Use este caminho no projeto ou se sobrar tempo. A lógica é a mesma do Caminho A; o que muda é a rede (a sua), a aplicação (um Apache simples) e a forma de gerar carga (K6 a partir do bastion, contra o ELB).

## B-1: Preparar a rede

Execute os **PASSOS 01 a 08** da instrução [ArquiteturaCorp](https://github.com/agodoi/ArquiteturaCorp/blob/main/README.md). Ao final desses passos, você terá a base da arquitetura, incluindo:

- **VPC\_Arquitetura\_Corp**
- sub-rede pública **Sub\_Publica\_a** em `us-east-1a`
- sub-rede privada **Sub\_Privada\_b** em `us-east-1b`
- tabela de rotas pública e tabela de rotas privada
- **NAT Gateway** permitindo saída para a internet a partir da sub-rede privada
- **Bastion\_Host\_Publica\_ArqCorp** (EC2 público)
- **EC2\_Privado\_ArqCorp** (EC2 privado)
- grupos de segurança **GS\_EC2Publico** e **GS\_EC2Privado**

Para usar um Application Load Balancer e um Auto Scaling Group distribuídos em **duas Zonas de Disponibilidade**, complete a rede antes de continuar:

**B-1.1)** Crie a segunda sub-rede pública:

- Nome: **Sub\_Publica\_b**
- Zona de Disponibilidade: `us-east-1b`
- CIDR IPv4: `192.168.2.0/24`
- Associe-a à tabela **TabRota\_Publica\_ArqCorp**, que possui a rota `0.0.0.0/0` para o Internet Gateway.

**B-1.2)** Crie a segunda sub-rede privada:

- Nome: **Sub\_Privada\_a**
- Zona de Disponibilidade: `us-east-1a`
- CIDR IPv4: `192.168.3.0/24`
- Associe-a à tabela **TabRota\_Privada\_ArqCorp**, que possui a rota `0.0.0.0/0` para o NAT Gateway.

**B-1.3)** Confirme que a VPC agora possui as quatro sub-redes usadas neste laboratório:

- **Sub\_Publica\_a**: `us-east-1a`
- **Sub\_Publica\_b**: `us-east-1b`
- **Sub\_Privada\_a**: `us-east-1a`
- **Sub\_Privada\_b**: `us-east-1b`

> Para esta prática, as duas sub-redes privadas podem usar a mesma tabela de rotas privada e o mesmo NAT Gateway. Em uma arquitetura de produção que também exija alta disponibilidade para a saída à internet, é comum utilizar um NAT Gateway por Zona de Disponibilidade.

## B-2: Instalar o Apache no EC2 privado

**B-2.1)** Conecte-se ao bastion por SSH e, a partir dele, ao EC2 privado.

**B-2.2)** No EC2 privado, instale o Apache:

```
sudo apt update
sudo apt install apache2 -y

```

**B-2.3)** Crie uma página simples com uma imagem, para facilitar a identificação visual da aplicação quando ela for acessada pelo navegador:

```
echo '<html><body><h1>Hello do EC2_Privado_ArqCorp!</h1><img src="logo.png"></body></html>' | sudo tee /var/www/html/index.html
sudo wget -O /var/www/html/logo.png https://www.inteli.edu.br/wp-content/uploads/2024/05/logo.png

```

**B-2.4)** Teste do próprio EC2 privado:

```
curl -s http://localhost | head

```

Deve retornar o HTML acima.

**B-2.5)** Digite `exit` para voltar ao bastion.

## B-3: Instalar o K6 no bastion

No bastion:

```
sudo apt update
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6 -y
k6 version

```

## B-4: Criar a AMI do EC2 privado

Siga o **Passo-01** do Caminho A, mas selecionando **EC2\_Privado\_ArqCorp** em vez de Web Server 1. Nome sugerido: **ApacheAMI**.

## B-5: Ajustar os grupos de segurança

Este é o ponto onde a arquitetura corporativa difere de verdade do laboratório. O ELB fica exposto; as instâncias, não.

**B-5.1)** Crie um grupo de segurança chamado **GS\_ELB** na VPC\_Arquitetura\_Corp com regra de entrada **HTTP (80) de 0.0.0.0/0**. Ele será usado **só pelo ELB**.

**B-5.2)** Edite o **GS\_EC2Privado** e adicione uma regra de entrada **HTTP (80) com origem = GS\_ELB**. Assim as instâncias só aceitam tráfego web vindo do balanceador. Mantenha a regra de SSH vinda do bastion.

> **Reflexão 6:** por que não abrir a porta 80 do GS\_EC2Privado para 0.0.0.0/0, já que as instâncias estão em sub-rede privada mesmo? (Dica: defesa em profundidade.)

## B-6: Criar grupo de destino e ELB

Siga o **Passo-02** do Caminho A com estas substituições:

| Item do Passo-02 Use      |                                                                  |
| ------------------------- | ---------------------------------------------------------------- |
| VPC (2.5 e 2.12)          | **VPC\_Arquitetura\_Corp**                                       |
| Sub-redes (2.13)          | **Sub\_Publica\_a** e **Sub\_Publica\_b** (as duas **públicas**) |
| Grupo de segurança (2.14) | **GS\_ELB** apenas                                               |
| Nomes                     | **CorpGroup** e **CorpELB**                                      |

## B-7: Criar modelo de execução e grupo do Auto Scaling

Siga o **Passo-03** do Caminho A com estas substituições:

| Item do Passo-03 Use               |                                                                    |
| ---------------------------------- | ------------------------------------------------------------------ |
| AMI (3.4)                          | **ApacheAMI**                                                      |
| Grupo de segurança do modelo (3.7) | **GS\_EC2Privado** apenas                                          |
| VPC (Etapa 2)                      | **VPC\_Arquitetura\_Corp**                                         |
| Sub-redes (Etapa 2)                | **Sub\_Privada\_a** e **Sub\_Privada\_b**                          |
| Grupo de destino (Etapa 3)         | **CorpGroup**                                                      |
| Tipo de métrica (Etapa 4)          | **Application Load Balancer request count per target**             |
| Recurso da métrica (Etapa 4)       | **CorpELB / CorpGroup**                                            |
| Valor de destino (Etapa 4)         | **1000 requisições por destino por minuto**                        |
| Nomes                              | **CorpConfig**, **Corp Auto Scaling Group**, **CorpScalingPolicy** |

> No Caminho B, usamos **ALBRequestCountPerTarget** em vez de CPU. O Apache serve uma página estática e pode consumir pouca CPU mesmo com muitas requisições. A contagem de requisições por destino está diretamente ligada à carga produzida pelo K6 e torna a demonstração de scale-out mais previsível. O valor `1000` é um valor didático para este laboratório e pode ser ajustado conforme o comportamento observado.

Depois, siga o **Passo-04** para confirmar que o DNS do **CorpELB** responde "Hello do EC2\_Privado\_ArqCorp!" no navegador.

## B-8: Teste de carga com K6 contra o ELB

O alvo do teste é o **DNS do ELB**, não o IP de uma instância. Só assim o tráfego passa pelo balanceador e o Auto Scaling entra em jogo.

**B-8.1)** No bastion, copie o nome do DNS do CorpELB e crie o script:

```
nano test.js

```

**B-8.2)** Cole o conteúdo abaixo, trocando `ELB-DNS` pelo nome do DNS do CorpELB:

```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export let options = {
  stages: [
    { duration: '1m', target: 200 },  // sobe até 200 usuários virtuais
    { duration: '5m', target: 200 },  // segura 200 por 5 min para manter carga suficiente durante a avaliação da política
    { duration: '1m', target: 0 },    // desce a zero
  ],
};

export default function () {
  let res = http.get('http://ELB-DNS/');
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
  sleep(1);
}

```

> O `http.get()` acima solicita apenas a página `/`. O K6 não funciona como um navegador e não baixa automaticamente a imagem `logo.png` referenciada pelo HTML. Para esta prática isso não é um problema, porque a política do Caminho B usa **requisições do ALB por destino**, e não o tamanho da página nem a utilização de CPU.

> Um teste muito curto pode terminar antes que a política tenha tempo de observar a métrica, decidir o scale-out, iniciar novas instâncias e registrá-las como íntegras. Por isso usamos um patamar de **5 minutos** de carga sustentada.

**B-8.3)** Salve com `Ctrl + X`, `Y`, `Enter`.

**B-8.4)** Execute:

```
k6 run test.js

```

**B-8.5)** Enquanto roda, o K6 mostra:

- **http\_req\_duration**: latência das respostas (observe o p95).
- **http\_reqs**: requisições por segundo que o sistema está processando.
- **checks**: percentual de respostas com status 200. Se cair abaixo de 100%, o servidor começou a rejeitar ou o ELB retornou 5xx.

**B-8.6)** Em paralelo, no CloudWatch, acompanhe o **AlarmHigh** do Corp Auto Scaling Group. Neste caminho, a métrica é **ALBRequestCountPerTarget**: o número médio de requisições por destino aumenta, a condição de scale-out é atendida, o alarme entra em estado **Em alarme** e novas instâncias aparecem. Em **EC2 > Grupos do Auto Scaling > Corp Auto Scaling Group > Atividade**, acompanhe as ações de lançamento.

**B-8.7)** Quando o K6 terminar, acompanhe o **scale-in** seguindo a mesma lógica apresentada na **Atividade complementar opcional: observar o scale-in** do Caminho A.

**B-8.8)** Experimente variar `target` (usuários virtuais) e a duração dos estágios. **Não passe de 1000 VUs** neste laboratório usando um bastion t2.micro, para evitar que o próprio gerador de carga se torne o gargalo ou fique sem recursos.

> **Reflexão 7:** compare a latência p95 do K6 antes e depois do scale-out. Ela melhorou? Se não melhorou, onde pode estar o gargalo (bastion, ELB, instâncias, tamanho t2.micro)?

## B-9: Encerrar

- Encerre o **EC2\_Privado\_ArqCorp** original (a AMI já foi gerada e as instâncias do grupo são independentes dele).
- Ao terminar o laboratório, **reduza a capacidade mínima e desejada do grupo para 0** ou exclua o grupo do Auto Scaling; caso contrário ele recria as instâncias sozinho.
- Exclua o ELB e o grupo de destino.

---

## Resumo do que você mediu

| Conceito Onde você viu                  |                           |
| --------------------------------------- | ------------------------- |
| AMI como cópia independente             | Passo-01 e Passo-06       |
| Health check e destino íntegro          | Passo-04                  |
| Balanceamento entre zonas               | Passo-02 e Passo-04       |
| Scale-out orientado por métrica         | Passo-05                  |
| Scale-in                                | Atividade complementar e B-8 |
| Segmentação público/privado             | Etapa 2 do Passo-03 e B-5 |
| Medição de carga (latência, RPS, erros) | B-8                       |
