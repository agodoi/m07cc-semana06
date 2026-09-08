# Computação Ágil e Elástica (parte prática - como medir)

Neste encontro iremos abordar o crescimento extremo e exponencial de acesso aos serviços e aplicações na nuvem e como sua arquitetura deve ser desenhada de forma a atender esta demanda crescente de acessos. Serão apresentados os 6 R"s da Migração e o Well-Architected Framework da AWS. Iremos compreender os conceitos de elasticidade, escalabilidade e alta disponibilidade no contexto da nuvem e como mitigar possíveis ataques como DDoS. Realizar testes de carga para verificar escalabilidade do sistema.

## Objetivos

* Subir 2 EC2: um bastion host e um EC2 como servidor interno
* Gerar a imagem do EC2 via auto scalling group (template)
* Adicionar o ELB
* Adicionar o serviço de Auto Scalling
* Instalar o K6 no bastion host
* Gerar tráfego do bastion para o EC2

## Impactos no seu projeto

Sua aplicação da Globoplay precisa de recursos de auto scalling e gestão na carga entre os computadores internos. A instrução de hoje ensina como fazer isso.

## Não faça

* Qualquer tentativa de ter 20 ou mais instâncias em execução simultânea (seja qual for o tamanho) causará a desativação imediata da conta da AWS e todos os recursos na conta serão excluídos imediatamente;

* Testes K6 usando o seu computador do Inteli para o servidor EC2 dentro da sua VPC;

* Testes de outro sites, como o do Inteli, pois vc será banido e vai arrumar um problemão;

## Conceitos

O [Elastic Load Balancer](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-ElasticLoadBalancer.md) distribui automaticamente o tráfego de entrada das aplicações por várias instâncias do Amazon EC2. Ele permite obter tolerância a falhas nos aplicativos por meio da disponibilização sem problemas da capacidade necessária de balanceamento de carga para rotear o tráfego de aplicativos.

O Auto Scaling ajuda a manter a disponibilidade da aplicação e permite aumentar ou reduzir a capacidade do Amazon EC2 de forma automática, de acordo com condições definidas por você. Você pode usar o Auto Scaling para ajudar a garantir que está executando o número desejado de instâncias Amazon EC2. 

O Auto Scaling também pode aumentar automaticamente o número de instâncias do Amazon EC2 durante picos de demanda para manter o desempenho, e diminuir a capacidade durante períodos ociosos para reduzir os custos. O Auto Scaling é ideal para aplicativos com padrões de demanda estáveis ou que passam por variações de utilização horárias, diárias ou semanais.

### Arquitetura Começa Assim:

<img src="https://github.com/agodoi/TesteCargaAWS/blob/main/imgs/starting-architecture.png" width="500">

<picture>
   <source media="(prefers-color-scheme: light)" srcset="https://github.com/agodoi/TesteCargaAWS/blob/main/imgs/starting-architecture.png">
   <img alt="Arquitetura Inicial" src="[YOUR-DEFAULT-IMAGE](https://github.com/agodoi/TesteCargaAWS/blob/main/imgs/starting-architecture.png)">
</picture>

### Arquitetura Finaliza Assim:

<picture>
   <source media="(prefers-color-scheme: light)" srcset="https://github.com/agodoi/TesteCargaAWS/blob/main/imgs/final-architecture.png">
   <img alt="Arquitetura Inicial" src="[YOUR-DEFAULT-IMAGE](https://github.com/agodoi/TesteCargaAWS/blob/main/imgs/final-architecture.png)">
</picture>

## Você tem 2 opções:

### OPC 1) Fazer um teste completo usando uma Arquitetura Corporativa. É provável que você use essa opção no seu projeto.

#### Caso escolha essa opção, comece do PASSO-00.

### OPC 2) Fazer o laboratório do Módulo 10 do Curso AWS Foundation. Vamos adotar essa opção para a aula.

#### Caso escolha essa opção, comece do PASSO-01.



__________________________________________________________________________________________________________________________________________________________________________________________
## Passo-00: Criar uma Rede Corporativa para Testes

Execute o **PASSO-01** até **PASSO-08** dessa instrução [https://github.com/agodoi/ArquiteturaCorp/blob/main/README.md](https://github.com/agodoi/ArquiteturaCorp/blob/main/README.md)

Quando terminar, volte aqui...

**0.1)** Entre no seu **Bastion_Host_Publica_ArqCorp** usando uma conexão ssh.

**0.2)** Entre no seu **EC2_Privado_ArqCorp** usando outra conexão ssh a partir do Bastion Host.

**0.3)** Instale um servidor simples no EC2_Privado, que será um Apache:

```
sudo apt update
sudo apt install apache2 -y
```

**0.4)** Crie uma página HTML simples no **index.html** e baixe um imagem simples lá.

```
echo '<html><body><h1>Hello do EC2_Privado_ArqCorp!</h1></body></html>' | sudo tee /var/www/html/index.html
```

Baixando uma imagem do site do Inteli:

```
sudo wget https://www.inteli.edu.br/wp-content/uploads/2024/05/logo.png
```

**0.5)** Saia do EC2 Privado e volte para Bastion Host. Vamos instalar o testador de carga K6. Digite **exit** na raiz do seu EC2 Privado.

**0.6)** Para aplicar um teste de carga utilizando o K6 no EC2 público (Bastion Host) para o EC2 privado que está rodando um servidor Apache, vamos instalar o K6 no Bastion:

```
sudo apt update
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6
```

## Passo-01: Criar uma AMI para o Auto Scaling

Nesta tarefa, você criará uma AMI com a instância Web Server 1 existente. Será o mesmo Web Server da instrução [https://github.com/agodoi/EC2-RDS](https://github.com/agodoi/EC2-RDS)

AMI é a sigla para Amazon Machine Image, ou Imagem de Máquina da Amazon, que é um tipo de dispositivo virtual fornecido pela AWS para criar uma máquina no Amazon Elastic Compute Cloud (EC2).

Isso salvará o conteúdo do disco de inicialização para que novas instâncias possam ser executadas com conteúdo idêntico.

**1.1)** Execute o laboratório do Módulo 10 do curso AWS Foundation. Vamos aproveitar esse laboratório para criarmos a arquitetura acima e depois, adicionar um Bastion Host e fazer os testes do K6.

**1.2)** No Console de Gerenciamento da AWS, na caixa de pesquisa ao lado de Serviços, pesquise e **escolha EC2**.

**1.3)** No painel de navegação à esquerda, selecione **Instâncias** e confirme que a instância está em execução. Aguarde até que as Verificações de status de Web Server 1 exibam 2/2 **verificações aprovadas** em verde. Se necessário, selecione “Atualizar” para atualizar o status.

Agora, você criará uma AMI com base nessa instância.

**1.4)** Selecione  **Web Server 1**.

**1.5)** No menu HORIZONTAL DO CONSOLE chamado **Ações**, selecione **Imagem e modelos** > **Criar imagem** e configure:

* Nome da imagem: **WebServerAMI**
* Descrição da imagem: **Lab AMI for Web Server**
* Deixei tudo como está sem alterar

**1.6)** Clique em **Criar imagem**. Um banner verde de confirmação exibe o ID (tipo ami-065d3b82f7a510b8e) da AMI nova. 

Você usará essa AMI ao iniciar o grupo do Auto Scaling posteriormente no laboratório. Guenta que nói chega lá.


## Passo-02: Criar um ELB

Nesta tarefa, primeiro você criará um [grupo de destino](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-GrupoDeDestino.md) e, depois, um balanceador de carga que pode balancear o tráfego entre várias instâncias do EC2 e Zonas de Disponibilidade.

Lembrando, que o Grupo de Destino é um subitem do ELB que serve para monitorar a integridade de EC2 de destinos que estão de pé, íntegros, registrados e prontos receber solicitações. [Clique aqui](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-GrupoDeDestino.md) para saber mais sobre Grupo de Destino.

**2.1)** No painel de navegação à esquerda, escolha **Grupos de destino** que está dentro de **Balanceamento de carga**.

**2.2)** Escolha **Criar grupo de destino**.

**2.3)** Escolha um tipo de destino: **Instâncias**.

**2.4)** Nome do grupo de destino, insira: **LabGroup** e deixa todos os itens sem alterar até a próxima etapa.

**2.5)** Selecione **Lab VPC** no menu suspenso VPC. Se você estiver usando a sua **VPC_Arquitetura_Corp**, selecione essa.

Análise: **grupos de destino** definem para qual local enviar o tráfego que entra no balanceador de carga. O [Application Load Balancer](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-ApplicationLoadBalancer.md) pode enviar tráfego para vários grupos de destino com base na URL da solicitação recebida, como ter solicitações de aplicativos móveis indo para outro conjunto de servidores. Neste exercício, o aplicativo web usará apenas um grupo de destino.

Dica: Na opção **Protocolo da verificação de integridade**, indica que o Grupo de Destino vai mandar um GET no servidor, e se voltar um **200 OK** e não os seguintes códigos do HTTP. Os códigos de status de resposta HTTP são agrupados em cinco classes: 

* Respostas Informativas (100 – 199)
* Respostas bem-sucedidas (200 – 299)
* Mensagens de redirecionamento (300 – 399)
* Respostas de erro do cliente (400 – 499)
* Respostas de erro do servidor (500 – 599)
  
**2.6)** Deixe tudo como está até aqui e selecione **Próximo**. A tela **Registrar destinos** é exibida.

Observação: **Destinos** são instâncias individuais que responderão às solicitações do balanceador de carga. Você ainda não tem nenhuma instância de aplicativo web rodando; portanto, pode ignorar esta etapa. **Caso você veja um Bastion Host ou algum outro EC2, ignore-o. NÃO MARQUE NADA! Siga em frente**.

**2.7)** Revise as configurações sem mexer em nada e selecione **Criar grupo de destino**.

**2.8)** No painel de navegação à esquerda, escolha **Balanceadores de carga** ou **Load Balancers**.

**2.9)** No topo da tela, selecione **Criar balanceador de carga**.

Vários tipos diferentes de balanceador de carga são exibidos:

**Application Load Balancer (ALB)**

* **Uso principal:** Distribui tráfego HTTP/HTTPS entre vários servidores, roteando com base em regras complexas (URL, cabeçalhos, etc.). Ideal para aplicações web modernas com microsserviços. Opera na Camada 7 (Aplicação) do modelo OSI.

**Network Load Balancer (NLB)**

* **Uso principal:** Distribui tráfego TCP/UDP de alta performance entre servidores, com baixo overhead. Ideal para aplicações que precisam de extrema velocidade e escalabilidade, como jogos online ou streaming de vídeo. Opera na Camada 4 (Transporte) do modelo OSI.

**Gateway Load Balancer (GLB)**

* **Uso principal:** Implanta e escala appliances de rede virtualizadas (firewalls, IDS/IPS, etc.) de forma transparente. O tráfego é direcionado para essas appliances antes de chegar aos servidores de aplicação. Opera na camada 3 (Rede) do modelo OSI.

Você usará um **Application Load Balancer** que opera no nível de solicitação (camada 7), roteando o tráfego para os destinos (instâncias do EC2, contêineres, endereços IP e funções do Lambda) com base no conteúdo da solicitação. Para saber mais, consulte Comparação de balanceadores de carga.

**2.10)** Em **Application Load Balancer**, selecione **Criar**.

**2.11)** Em **Nome** do balanceador de carga, insira: **LabELB**.

**2.12)** Role para baixo até a seção [**Mapeamento de rede**](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-MapeamentoRede.md) e, depois, em **VPC**, selecione **Lab VPC** ou **VPC_Arquitetura_Corp** se você estiver aproveitando sua infra do Learner Lab.

Agora você especificará quais sub-redes o balanceador de carga deve usar. Será um balanceador de carga voltado para a Internet; portanto, selecione as duas sub-redes públicas, mas calma que vamos fazer por partes para não dar pau no final.

**2.12.1)** Escolha a **primeira** Zona de Disponibilidade exibida e selecione **Sub-rede pública 1** no menu suspenso **Sub-rede** exibido abaixo dela. Ou se estiver na ArquiteturaCorp a **Sub_Publica_a** e **Sub_Privada_b**.

**2.12.2)** Escolha a **segunda** Zona de Disponibilidade exibida e selecione **Sub-rede pública 2** no menu suspenso **Sub-rede** exibido abaixo dela.

Agora você terá duas sub-redes selecionadas: **Sub-rede pública 1** e **Sub-rede pública 2**. 
 

**2.13)** Na seção **Grupos de segurança:**, escolha **Grupos de segurança** no menu suspenso e selecione [**Web Security Group**](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-WebSecurityGroup.md). Ou **GS_EC2Privado** + **GS_EC2Publico**.

Abaixo do menu suspenso, selecione o **X** ao lado do grupo de segurança **Default** e clique para removê-lo.

O grupo de segurança **Web Security Group** (Grupo de segurança da web) agora deve ser o único que aparece.

**2.14)** Para a linha **Listener HTTP:80**, defina a ação padrão para encaminhar para **LabGroup**. Lembra disso? É o seu **Grupo de Destino HTTP**. Mas se nessa hora estiver vazio, deu ruim na etapa **2.4**, volte para o início do PASSO-02 e refaça tudo.

**2.15)** Role para baixo, deixe tudo como está.

**2.16)** Selecione **Criar balanceador de carga**. 

**2.17)** Aguarde e o balanceador de carga terá sido criado com sucesso.


## Etapa 03 - Criar um modelo de inicialização e um grupo do Auto Scaling

Nesta tarefa, você criará um [modelo de execução](https://github.com/agodoi/m07-semana06/blob/main/doc/definicao-ModeloDeExecucao.md) para seu grupo do Auto Scaling.

**3.1)** No painel de navegação à esquerda, selecione **Modelos de execução** que está em **Instâncias**.

**3.2)** Selecione **Criar modelo de execução**.

**3.3)** Defina as configurações do modelo de execução e crie-o:

**3.3.1)** Nome do modelo de execução: **LabConfig**

**3.3.2)** Em **Auto Scaling guidance** (Orientação sobre o Auto Scaling), selecione **Provide guidance to help me set up a template that I can use with EC2 Auto Scaling** ou **Fornecer orientação para me ajudar a configurar um modelo que eu possa usar com o EC2 Auto Scaling**.

**3.3.3)** Na área **Application and OS Images** (Amazon Machine Image) (Imagens da aplicação e do SO [Imagem de Máquina da Amazon]), selecione **Minhas AMIs**.

**3.3.4)** Imagem de máquina da Amazon (AMI): escolha **Web Server AMI** (AMI do servidor web).

**3.3.5)** Tipo de instância: selecione **t2.micro** (que é o nível gratuito).

**3.3.6)** Nome do par de chaves: selecione **vockey**.

**3.3.7)** Em **Sub-rede** você não mexe e em **Firewall** (grupos de segurança), confirme se está marcado **Selecionar grupo de segurança existente**.

**3.3.8)** Grupos de segurança: escolha **Web Security Group** (Grupo de segurança da web). Caso esteja usando a Arquitetura Corporativa, marque **GS_EC2Publico** + **GS_EC2Privado**.

**3.3.9)** Role para baixo até a área **Detalhes avançados** e expanda-a.

**3.3.10)** Role para baixo até a configuração **Monitoramento do CloudWatch detalhado**. Selecione **Habilitar**. Nota: isso permitirá que o Auto Scaling reaja rapidamente a alterações na utilização.

**3.3.11)** Deixe todo o restante do jeito que está e clique no botão laranja **Criar modelo de execução**. A seguir, você criará um grupo do Auto Scaling que usa esse modelo de execução.

**3.4)** Na caixa de diálogo de tarja verde **Êxito**, selecione o modelo de execução **LabConfig**.

**3.5)** No menu **Ações**, selecione **Criar grupo do Auto Scaling**.

### Presta a atenção à esquerda que você está configurando ETAPAS 1 a 7...

### ETAPA 1

**3.6)** Configure os detalhes na **Etapa 1** (Selecione o modelo de execução ou a configuração):

**3.6.1)** Nome do grupo do Auto Scaling: **Lab Auto Scaling Group**.

**3.6.2)** Modelo de execução: confirme se o modelo **LabConfig** que você acabou de criar foi selecionado.

**3.6.3)** Deixe o restante como está e selecione **Próximo**.

### ETAPA 2

**3.7)** Configure os detalhes na **Etapa 2** (Selecione as opções para executar a instância):

**3.7.1)** VPC: selecione **Lab VPC** ou **VPC_Arquitetura_Corp**.

**3.7.2)** Zonas de disponibilidade e sub-redes: escolha **Sub-rede privada 1** e **Sub-rede privada 2**. Agora vc está arrumando a rede interna, porque a rede exeterna foi resolvida na etapa anterior.

**3.7.3)** Selecione **Próximo**.

### ETAPA 3

**3.8)** Configure os detalhes na **Etapa 3** (Configure as opções avançadas):

**3.8.1)** Selecione **Anexar a um balanceador de carga existente** (caixinha do meio). Grupos de destino de balanceador de carga existentes: selecione **LabGroup | HTTP**.

**3.8.2)** Mais para baixo, encontre o título **Configurações adicionais**, selecione **Enable group metrics collection within CloudWatch** ou **Habilitar coleta de métricas de grupo no CloudWatch**. Essa ação captura métricas em intervalos de um minuto, o que permite que o Auto Scaling reaja rapidamente a mudanças nos padrões de uso. Selecione **Próximo**.

### ETAPA 4

**3.9)** Configure os detalhes na **Etapa 4** (Configure o tamanho do grupo e as políticas de scaling: opcional). Mexa apanas onde essa instrução lhe pede para mexer.

**3.9.1)** Em **Tamanho do grupo**, configure: 

**3.9.2)** Capacidade desejada: 2

**3.9.3)**    Capacidade mínima: 2

**3.9.4)** Capacidade máxima: 6. 

Isso permitirá que o Auto Scaling adicione/remova instâncias automaticamente, mantendo sempre de 2 a 6 instâncias em execução. **LEMBRE-SE DA REGRA DOS 20 EC2, PELO AMOR DE DEUS!***

**3.9.5)** Em **Escalabilidade**, escolha a caixinha **Política de escalabilidade de rastreamento de destino** ou **Política de dimensionamento com monitoramento do objetivo** e configure:

**3.9.6)** Nome da política de escalabilidade: **LabScalingPolicy**.

**3.9.7)** Tipo de métrica: **Média de utilização da CPU**.

**3.9.8)** Valor de destino: **60**. Isso informa ao Auto Scaling para manter uma utilização média de CPU em todas as instâncias em 60%. O Auto Scaling adicionará ou removerá automaticamente a capacidade conforme necessário para manter a métrica no valor de destino especificado ou próxima dele. Ele se ajusta às flutuações na métrica devido a um padrão de carga flutuante.

**3.9.9)** Não mexa em mais nada e selecione **Próximo**.

### ETAPA 05

**3.10)** Configure os detalhes na **Etapa 5** (Adicione notificações: opcional): o Auto Scaling pode enviar uma notificação quando ocorre um evento de scaling. Você usará as configurações padrão. Para essa aula, não mexa em nada e selecione **Próximo**, mas você poderá fazer isso no seu projeto depois.

### Agora você está na ETAPA 06

**3.11)** Configure os detalhes na **Etapa 6** (Adicione tags: opcional): as tags aplicadas ao grupo do Auto Scaling serão propagadas automaticamente para as instâncias executadas. 

**3.11.1)** Escolha **Adicionar tag** e configure o seguinte:

**3.11.2)** Chave: **Name**

**3.11.3)** Valor: **Lab Instance**

**3.11.4)** Selecione **Próximo**.

### ETAPA 07

**3.12)** Nessa etapa não faremos nada. Só observe o que foi feito e escolha **Criar grupo do Auto Scaling**. O grupo do Auto Scaling mostrará inicialmente uma contagem de instâncias igual a zero, mas novas instâncias serão executadas para atingir a contagem desejada de **duas instâncias**.

## Passo-04: Verificar se o balanceamento de carga está funcionando

Nesta tarefa, você verificará se o balanceamento de carga está funcionando corretamente.

**4.1)** No painel de navegação à esquerda, selecione **Instâncias**. Devem aparecer duas novas instâncias chamadas **Lab Instance**. Elas foram iniciadas pelo Auto Scaling. Se as instâncias ou nomes não forem exibidos, aguarde 30 segundos e selecione **Rodinha Skoll desce redondo**  no canto superior direito. Em seguida, você confirmará que as novas instâncias foram aprovadas na health check.

**4.2)** No painel de navegação à esquerda, escolha **Grupos de destino** e entre no **LabGroup** clicando em seu link azul.

**4.2.1)** Pressione a **rodinha Skoll desce redondo** para atualizar a sua tela.

**4.3)** Escolha o menu horizontal **Destinos**. Duas instâncias de destino chamadas **Lab Instance** (Instância do laboratório) devem ser listadas no grupo de destino. Se estiver vazio, **rodinha da Skoll desce redondo** para atualizar novamente.

**4.3)** Aguarde até que o **Status** de ambas as instâncias mude para íntegro. Selecione **Atualizar** no canto superior direito para verificar se há atualizações, caso necessário.

O status íntegro indica que a **instância passou na health check do balanceador de carga**. Isso significa que o balanceador de carga enviará tráfego para a instância. Agora você pode acessar o grupo do Auto Scaling por meio do balanceador de carga.

**4.4)** No painel de navegação à esquerda, escolha **Balanceadores de carga**.

**4.5)** Selecione o balanceador de carga **LabELB** pelo seu link azul.

**4.6)** No painel **Detalhes**, copie o **Nome do DNS** (não é o **ARN do load balancer** e sim o endereço à direita dele, algo do tipo **LabELB-1152052616.us-east-1.elb.amazonaws.com**) do balanceador de carga, omitindo “(Registro A)”.

**4.7)** Abra uma nova guia do navegador da web, cole o nome do DNS que você acabou de copiar e pressione Enter. O aplicativo deve aparecer em seu navegador. Você deve se lembrar desse app porque você já fez uma agenda com ele.

## Se você chegou até aqui, Parabéns!

Isso indica que o balanceador de carga recebeu a solicitação, a enviou para uma das instâncias do EC2 e, em seguida, repassou o resultado.

## Passo-05: Testar o Auto Scaling

Você criou um grupo de Auto Scaling com um mínimo de duas instâncias e um máximo de seis instâncias. Atualmente, duas instâncias estão em execução porque o tamanho mínimo é duas e o grupo não está atualmente sob nenhuma carga. Agora, você aumentará a carga para fazer com que o Auto Scaling acrescente outras instâncias.

**5.1)** Volte para o Console de Gerenciamento da AWS, mas não feche a guia da aplicação. Você retornará a ela em breve.

**5.2)** Na caixa de pesquisa ao lado de **Serviços**, pesquise e selecione **CloudWatch**. Deixe uma Estrelinha marcada que tem ao lado dele que você vai criando seu atalho personalizado.
 
**5.3)** No painel de navegação à esquerda, expanda **Alarmes** e selecione **Todos os alarmes**.

Dois alarmes serão exibidos. Foram criados automaticamente pelo grupo de Auto Scaling. Manterão automaticamente a carga média da CPU próxima a 60%, permanecendo também dentro da limitação de ter de 2 a 6 instâncias.

### ATENÇÃO: siga estas etapas somente se você não vir os alarmes em 60 segundos ou quanto tiver pressionado o Skoll desce redondo.


**5.4)** No menu **Serviços**, selecione **EC2**.

**5.5)** No painel de navegação esquerdo, escolha **Grupos do Auto Scaling** (um do últimos do menu vertical esquerdo).

**5.6)** Selecione **Lab Auto Scaling Group** (Grupo do Auto Scaling do laboratório).

**5.7)** No menu horizontal na metade da página, escolha a guia **Auto Scaling** ou **Escalabilidade Automática**.

**5.8)** Selecione a caixinha **LabScalingPolicy**. Você vai ver que tem onde ticar nessa opção.   

**5.9)** Selecione **Ações** e **Editar**.

**5.10)** Altere o **Valor de destino** para **50**. Selecione **Atualizar**.

**5.11)** No menu **Serviços**, selecione **CloudWatch**.

**5.12)** No painel de navegação à esquerda, selecione **Todos os alarmes** e confirme se há dois alarmes.

**5.13)** Selecione o alarme **OK**, que tem **AlarmHigh** no nome. Se nenhum alarme estiver mostrando OK, aguarde um minuto e selecione **Atualizar** no canto superior direito até que o status do alarme mude. Mas atenção: **OK** indica que o alarme **NÃO** foi acionado. É o alarme para **Utilização de CPU > 60** (lembra dos 60% de capacidade computacional criado no passo **3.9.8**?), que adicionará instâncias quando a CPU média estiver alta. O gráfico deve mostrar níveis muito baixos de CPU no momento. Agora, você informará ao aplicativo para executar cálculos que devem aumentar o nível de CPU.

**5.14)** Volte para a guia do navegador com o aplicativo web.

**5.15)** Selecione **Load Test** (Teste de carga) ao lado do logotipo da AWS. Isso fará com que o aplicativo gere cargas elevadas. A página do navegador será atualizada automaticamente para que todas as instâncias no grupo do Auto Scaling gerem carga. Não feche esta guia.

**5.16)** Volte para a guia do navegador com o console do **CloudWatch**. Em menos de cinco minutos, o alarme **AlarmLow** deverá mudar para **OK** e o status do alarme **AlarmHigh** deverá mudar para **Em alarme**. Você pode selecionar **Atualizar** no canto superior direito a cada 60 segundos para atualizar a exibição. Você deve ver o gráfico **AlarmHigh** indicando uma porcentagem crescente de CPU. Depois de cruzar a linha de 60% por mais de 3 minutos, o Auto Scaling acionará a adição de instâncias adicionais.

**5.17)** Aguarde até que o alarme **AlarmHigh** entre no estado em vermelho **Em alarme**. Agora você pode visualizar as instâncias adicionais que foram executadas.

**5.18)** Na caixa de pesquisa ao lado de **Serviços**, pesquise e selecione **EC2**.

**5.14)** No painel de navegação à esquerda, selecione **Instâncias**. Agora deve haver mais de duas instâncias rotuladas como **Lab Instance** em execução. As instâncias novas foram criadas pelo Auto Scaling em resposta ao alarme CloudWatch.

### Conclusão: sua estrutura está em pleno autoscalling. Esse é um dos pontos chaves da AWS!
A aplicação http://labelb-1152052616.us-east-1.elb.amazonaws.com/load.php não encerra o auto teste de carga. Portanto, não vamos ver o EC2 sendo "devolvido".


## Passo-06: Carga de Teste K6

### Só faça esse passo caso você tenha optado pelo OPC (1) Arquitetura Corporatativa para Testes.
### Caso tenha optado pelo OPC (2), pule para o Passo-07


**6.1)** Vamos criar um script K6 de teste. No Bastion Host, crie um arquivo chamado `test.js`. Mas antes, pegue o IP local do EC2 Privado e anote aí num canto.

```
nano test.js
```

**6.2)** Cole o seguinte conteúdo, que faz requisições GET ao servidor Apache na instância privada e altere o **private-ip** no script que veio do EC2 Privado.

```
import http from 'k6/http';
import { sleep, check } from 'k6';

export let options = {
   vus: 1000, // Número de usuários virtuais
   duration: '30s', // Duração do teste
};

export default function () {
   let res = http.get('http://private-ip');
   check(res, {
       'status is 200': (r) => r.status === 200,
   });
   sleep(1);
}
```
#### Substitua `private-ip` pelo IP privado da instância EC2 privada onde o Apache está rodando.

**6.3)** Após colar o código e alterar o IP, pressione `Ctrl + X`, depois `Y` e `Enter` para salvar e sair.

**6.4)** Com o script pronto, você pode executar o teste no seu Bastion Host

```
k6 run test.js
```

O K6 começará a enviar requisições para o servidor Apache na instância privada. Você verá os resultados do teste em tempo real, incluindo a taxa de requisições por segundo, as latências e outras métricas.

**6.5)** Enquanto o teste estiver rodando, o K6 mostrará na tela estatísticas como:

- **HTTP Status Codes**: Quantos pedidos foram bem-sucedidos (status 200).
- **Request Duration**: Tempo que levou para receber as respostas.
- **Requisições por segundo**: Quantidade de requisições que o servidor está conseguindo processar.

**6.6)** Dependendo do que você deseja testar, você pode ajustar os parâmetros `vus` (usuários virtuais) e `duration` (duração do teste) no script `test.js`. Por exemplo, aumentar `vus` para simular mais usuários simultâneos ou aumentar `duration` para ver como o servidor se comporta ao longo de um período mais longo. Se colocar vus = 10000, vai travar o seu EC2 Bastion Host e terá que reiniciá-lo.

**6.7)** Volte para a guia do navegador com o console do **CloudWatch**. Analise o status dos alarmes em vermelho e explore os gráficos que aparecem por lá, informando a carga computacional que os EC2 sofreram. 

Em menos de cinco minutos, o alarme **AlarmLow** deverá mudar para **OK** e o status do alarme **AlarmHigh** deverá mudar para **Em alarme**. Você pode selecionar **Atualizar** no canto superior direito a cada 60 segundos para atualizar a exibição. Você deve ver o gráfico **AlarmHigh** indicando uma porcentagem crescente de CPU. Depois de cruzar a linha de 60% por mais de 3 minutos, o Auto Scaling acionará a adição de instâncias adicionais.

## Passo-07: Encerrar a instância Web Server 1

Nesta tarefa, você encerrará a instância **Web Server 1**. Essa instância foi utilizada para criar a AMI usada por seu grupo do Auto Scaling, mas ela não é mais necessária.

**7.1)** Selecione  Web Server 1 e verifique se é a única instância selecionada. Se você desligar o Web Server 1, as **cópias** Lab Instance serão devolvidas.

**7.2)** No menu **Estado da instância**, selecione **Estado da instância** > **Encerrar instância**.

Selecione **Encerrar**.


## Envio do trabalho. Esse lab é do módulo 10

* Para registrar seu progresso, selecione **Enviar** no topo destas instruções.

* Quando solicitado, selecione **Sim**.

* Depois de alguns minutos, o painel de notas é exibido e mostra quantos pontos você obteve em cada tarefa. Se os resultados não forem exibidos após alguns minutos, selecione **Notas** no topo destas instruções. Dica: você pode enviar seu trabalho várias vezes. Depois de fazer as alterações, selecione **Enviar** novamente. Seu último envio é registrado para o laboratório.

* Para ver o feedback detalhado sobre seu trabalho, selecione **Relatório de envio**. Dica: se você não tiver recebido a pontuação total em alguma verificação, poderá haver detalhes úteis no relatório de envio.

## Laboratório concluído

Parabéns! Você concluiu um dos laboratórios mais detalhado do curso.

* Selecione **Encerrar laboratório** no topo da página e **Sim** para confirmar que você deseja encerrar o laboratório.

* Um painel será exibido com a mensagem: "DELETE has been initiated... (A EXCLUSÃO foi iniciada...) você pode fechar esta caixa de mensagem agora.”

* Selecione o X no canto superior direito para fechar o painel.
