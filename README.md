# VmBix [![Build Status](https://travis-ci.org/dav3860/vmbix.svg?branch=master)](https://travis-ci.org/dav3860/vmbix)

VmBix é um proxy TCP multi-threaded para o VMWare Sphere API escrito em Java. Ele aceita conexões de um Zabbix servidor / proxy / agente ou o binário zabbix_get e traduz-los para chamadas de API VMware.

A partir da versão 2.2, o Zabbix pode monitorar nativamente um ambiente VMWare. Mas há algumas desvantagens:

Os itens monitorados não são todos muito relevantes
Isso não é facilmente extensível
Os hosts ESX e VM criados são na sua maioria somente de leitura. Não é possível anexá-los a modelos diferentes, colocá-los em grupos diferentes ou usar um agente Zabbix para monitorar seus sistemas operacionais ou aplicativos
VmBix ajuda você a superar essas limitações, com um desempenho muito bom. É multi-threaded, implementa objetos de cache, e pode ser consultado usando um Zabbix módulo carregável .

VmBix vem com um conjunto de modelos adicionando vários itens monitorados, gatilhos e gráficos no Zabbix. Aqui estão alguns screenshots do que você pode esperar em Zabbix: 
![](https://github.com/dav3860/vmbix/blob/master/screenshots/latest_data.png)

![](https://github.com/dav3860/vmbix/blob/master/screenshots/triggers.png)

![](https://github.com/dav3860/vmbix/blob/master/screenshots/graph.png)

Você pode usar métodos VmBix para consultar métricas interessantes de VMWare, por exemplo:


```
esx.counter[esx01.domain.local,cpu.ready.summation]
1135
```

```
vm.counter.discovery[VM01,virtualDisk.totalReadLatency.average]
{
    "data": [
        {
            "{#METRICINSTANCE}": "scsi2:2"
        },
        {
            "{#METRICINSTANCE}": "scsi2:1"
        },
        {
            "{#METRICINSTANCE}": "scsi2:0"
        },
        {
            "{#METRICINSTANCE}": "scsi2:6"
        },
        {
            "{#METRICINSTANCE}": "scsi2:5"
        },
        {
            "{#METRICINSTANCE}": "scsi2:4"
        },
        {
            "{#METRICINSTANCE}": "scsi2:3"
        }
    ]
}
```

```
vm.counter[VM01,virtualDisk.totalReadLatency.average,scsi2:4,300]
2
```

## Instalação
Obter a versão mais recente do servidor aqui . RPM & DEB pacotes são fornecidos.

Você também vai precisar do Zabbix módulo carregável correspondente à sua versão zabbix. Veja https://github.com/dav3860/vmbix_zabbix_module para detalhes de instalação.

O servidor VmBix pode ser instalado na mesma máquina que um servidor ou proxy Zabbix. O módulo carregável deve ser instalado na máquina Zabbix que monitorará o ambiente VMWare.

## Or build from source
Nota: você precisará instalar o JDK eo Maven para compilar VmBix.
* Install Maven

Follow the instructions on [this](https://maven.apache.org/install.html) page to install Maven.
* Compile
```
tar -zxvf vmbix-x.x.tar.gz
cd vmbix-x.x
mvn package
```

See the instructions [here](https://github.com/dav3860/vmbix_zabbix_module) to compile the loadable module.

## Inicio rápido

Note: to run VmBix you'll have to install a JRE (OpenJDK should suite but not tested). All the scripts are tested on Centos 7 but should work on other \*NIX distributions as well.

### Test the binary

Try to run /usr/local/sbin/vmbix, you should get a *usage* output :

```
# /usr/local/sbin/vmbix
Usage:
vmbix {-P|--port} listenPort {-s|--serviceurl} http[s]://serveraddr/sdk {-u|--username} username {-p|--password} password [-f|--pid pidfile] [-i|--interval interval] [-U|--useuuid (true|false)]
or
vmbix [-c|--config] config_file  [-f|--pid pidfile] [-i|--interval interval] [-U|--useuuid (true|false)]
```

### Configure the daemon

É altamente recomendável verificar os parâmetros antes de os escrever num ficheiro de configuração. Execute VmBix com seu nome de usuário, senha e URL de serviço. É recomendável usar um usuário somente leitura para se conectar ao vCenter.

```
$ vmbix -P 12050 -u "MYDOMAIN\\myvmwareuser" -p "mypassword"" -s "https://myvcenter.mydomain.local/sdk"
log4j:WARN No appenders could be found for logger (com.vmware.vim25.ws.XmlGenDom).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
17:39:08.430 [main] INFO net.dav3860.VmBix - starting server on port 12050
17:39:08.433 [main] INFO net.dav3860.VmBix - server started

```
Você deve ver uma saída semelhante. Depois de validar que você pode iniciar o VmBix, você pode editar o arquivo de configuração /etc/vmbix/vmbix.conf e executar o daemon. Você ainda pode executar o processo em primeiro plano para solução de problemas em tempo real, se necessário.

Para instalar como um daemon:
```
chkconfig --add vmbixd
```
Now you may start the daemon:
```
service vmbixd start
```
And configure autostart if you wish:
```
chkconfig vmbixd on
```
For logs, check the file :
```
tail -f /var/log/vmbix.log
```

### Configurar o Zabbix

Todos os servidores ESX, datastores e máquinas virtuais serão automaticamente descobertos e criados no Zabbix. Veja as instruções a seguir para configurar o Zabbix.

#### Importar os modelos
Importar os modelos a partir daqui ou de / usr / share / vmbix / zabbix / templates Se você instalou um pacote (importar o modelo vCenter depois que os outros). No momento, apenas os modelos Zabbix 3.0.x são fornecidos. Os itens VmBix no Zabbix são configurados com um tipo de "verificação simples", pois o Zabbix usa um módulo carregável para falar com o VmBix. Portanto, ainda é possível usar um agente Zabbix em paralelo para monitorar os hosts. O vmbix.so módulo carregável deve ser instalado no seu servidor / proxy.

#### Descubra os objetos
VmBix pode descobrir e criar seu ambiente de VMWare (hypervisors, VMs, datastores) em maneiras demais:
- Usando o Zabbix Low-Level Discovery (LLD) e protótipos de host
- usando um script fornecido falar com a API Zabbix para criar exércitos regulares Zabbix ( recomendado )

#### Usando o Zabbix LLD
Criar um host chamado "VmBix", por exemplo, e vinculá-lo com o VmBix vCenter modelo. O endereço IP ea porta não são utilizados, mas é necessário fazê-lo monitorado pelo servidor / proxy executando o módulo carregável.

Aguarde até que os servidores ESX, datastores e máquinas virtuais sejam descobertos e criados. Eles serão automaticamente vinculados ao VmBix ESX, datastore ou modelo VM. Talvez seja necessário aumentar o parâmetro Timeout no arquivo de configuração Zabbix se o VSphere demorar muito para responder.

Você também pode vincular modelos adicionais aos hosts criados editando o protótipo de host correspondente nas regras de descoberta de modelos VmBix vCenter.

Como esses hosts são criados usando o mecanismo de protótipo do host no Zabbix, eles serão quase de leitura. Por exemplo, você não pode editar um host para vinculá-lo a um modelo específico. Isso deve ser feito no nível do protótipo do host, o que pode ser uma limitação se suas máquinas virtuais forem diferentes.

#### Usando objetos VMWare como hosts regulares no Zabbix
ara superar essa limitação, você pode desativar a regra de descoberta de VM no modelo VmBix vCenter e criar suas máquinas virtuais manualmente no Zabbix. Em seguida, vinculá-los ao modelo VmBix VM (preferencialmente com o método de módulo carregável). Você pode editá-los como qualquer outro host.

Nota: se o useuuid parâmetro é definido como verdadeiro no arquivo de configuração VmBix, os objetos devem ser referenciados usando seu UUID VMWare. Portanto, se você criar um host manualmente, você deve definir seu nome para o UUID e seu nome visível para o nome da VM. Você pode usar os métodos * .discovery [*] para obter o UUID de um objeto:

```
# zabbix_get -s 127.0.0.1 -p 12050 -k "vm.discovery[*]"
{
  "data": [
    {
      "{#VIRTUALMACHINE}": "MYVM01",
      "{#UUID}": "4214811c-1bab-f0fb-363b-9698a2dc607c"
    },
    {
      "{#VIRTUALMACHINE}": "MYVM02",
      "{#UUID}": "4214c939-18f1-2cd5-928a-67d83bc2f503"
    }
  ]
}
```

Como seria uma dor para criar todas as suas máquinas virtuais / ESX / datastores manualmente, um exemplo de script de importação (vmbix-objeto-sync) é fornecido para esta finalidade. Esta é a maneira recomendada para descobrir seu ambiente com o VmBix. Verifique as instruções aqui para instalar e configurar o script.

Se você instalou o VmBix de um pacote, o script está localizado em /usr/share/vmbix/zabbix/addons.

### Consultando VmBix na CLI
Você pode consultar VmBix como um agente Zabbix usando a ferramenta zabbix_get:
```
# zabbix_get -s 127.0.0.1 -p 12050 -k about[*]
VMware vCenter Server 5.1.0 build-1364037
# zabbix_get -s 127.0.0.1 -p 12050 -k esx.status[esx01.domain.local]
1
# zabbix_get -s 127.0.0.1 -p 12050 -k vm.guest.os[4214811c-1bab-f0fb-363b-9698a2dc607c]
CentOS 4/5/6 (64 bits)
# zabbix_get -s 127.0.0.1 -p 12050 -k esx.discovery[*]
{
  "data": [
    {
      "{#ESXHOST}": "esx01.domain.local"
    },
    {
      "{#ESXHOST}": "esx02.domain.local"
    }
  ]
}# zabbix_get -s 127.0.0.1 -p 12050 -k vm.counter[MYVM01,virtualDisk.totalReadLatency.average,scsi0:1,300]
2
```
Novamente, se useuuid for definido como true no arquivo de configuração, os objetos serão identificados usando seu UUID:
```
# zabbix_get -s 127.0.0.1 -p 12050 -k vm.guest.os[421448c4-8970-28f0-05a5-90a20724bd08]
CentOS 4/5/6 (64 bits)
```

## Verificações suportadas do Zabbix
Há uma descrição completa do VmBix suportado métodos na [Wiki](https://github.com/dav3860/vmbix/wiki) section.

## Caching
Para desativar as variáveis CacheTtl do conjunto de cache para 0.

## Como implementar seus próprios cheques
1. Encontrar uma função chamada
```
private void checkAllPatterns                (String string, PrintWriter out  )
```
2.Adicione seu próprio teste padrão. Por exemplo, esta seqüência:
```
Pattern pHostCpuUsed            = Pattern.compile("^(?:\\s*ZBXD.)?.*esx\\.cpu\\.load\\[(.+),used\\]"             );        // :checks host cpu usage
```
Será responsável por este item:
```
esx.cpu.load[{HOST.DNS},used]
```
3. Role para baixo até o próximo bloco de código na mesma função começando com "String found;", adicione o seu próprio bloco "found =":
```
found = checkPattern(pHostCpuUsed           ,string); if (found != null) { getHostCpuUsed           (found, out); return; }
```
Este chama uma função chamada "getHostCpuUsed" com {HOST.DNS} como um primeiro argumento e uma instância PrintWriter como um segundo.

4. Sua função deve aceitar argumentos String e PrintWriter. Ele deve retornar valores como esse:
```
out.print(value);
out.flush();
```
5. Adicione uma linha de uso à função methods ():
```
static void methods() {
        System.err.print(
            "Available methods :                                  \n"
            [...]
          + "esx.cpu.load[name,used]                              \n"
```

## Consultando vários vCenters
No momento, VmBix não suporta vários vCenters. Se você ainda quiser consultar vários vCenters, será necessário instalar o VmBix em diferentes proxies Zabbix, apontando para vCenters diferentes. Em seguida, selecione o proxy certo em cada página de configuração do host Zabbix.
- Se você usar Zabbix LLD para descobrir o ambiente VMWare, você precisará criar vários hosts VmBix, um para cada instalação de proxy / VmBix.
- Se você usar o Python [script](https://github.com/dav3860/vmbix/tree/master/zabbix/addons),você terá que editar o arquivo de configuração para atribuir um proxy Zabbix para os anfitriões descobertos.

## Version history
See [CHANGELOG](https://github.com/dav3860/VmBix/blob/master/CHANGELOG.md)


This project is a fork of the original VmBix by [@hryamzik](https://github.com/hryamzik).
