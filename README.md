# node-problem-detector + kured

Este repositório guarda uma implementação customizada de duas checagens para o **node-problem-detector**, uma baseda em log e outra em uma resposta da rede. A intenção é que o node-problem-detector avise o **kured** a respeito da necessidade de uma reinicialização.

Segundo a documentação do kured a reinicialização acontece imediatamente após a criação do arquivo `/var/run/reboot-required`. Configurar `--period=2h` no **DaemonSet** é tempo suficiente para a equipe entrar em ação.
Uma boa ideia seria escolher um outro diretório através de `--reboot-sentinel` para estes arquivos, já que expondo `/var/run` dentro dos contêineres a API do Docker ficará exposta.

Instalação do kured:

	kubectl apply -f https://github.com/weaveworks/kured/releases/download/1.4.4/kured-1.4.4-dockerhub.yaml

Instalação do node-problem-detector (com base neste repositório):

	kubectl apply -f rbac.yml -f node-problem-detector-config.yaml -f node-problem-detector.yaml

As configurações customizadas estão em `node-problem-detector-config.yaml`.

Cada checagem customizada possui suas **conditions** e suas **rules** definidas em um array. A **condition** indica a mensagem que aparecerá por padrão no node:

```
Conditions:
Type                 Status    ...   Reason                       Message
----                 ------          ------                       -------
LogFlooding          False     ...   LogIsNotFlooding             Log file is growing health
PodAnswerTimeout     False     ...   PodIsReachable               Node can reach the pod
```

As rules de tipo **temporary** são temporárias e aparecerão apenas nos **events** de cada node. Já as rules de tipo **permanent** fazem referência a um **condition** pelo **type** dos conditions declarados, estes casos são considerados permanentes e aparecerão nas condições do node, indicando um problema.

> É a inspeção dos **status** dos **conditions** que indicarão o problema, nos exemplos do repositório o status **True** indica um problema.

## Log

Este script extrai o tamanho do diretório de log e o guarda em `/tmp/log-last-check`. Após a primeira verificação passa a comparar o último tamanho com o tamanho atual, se a diferença for maior que 250 bytes começa a emitir o sinal para o **node-problem-detector**.

```bash
#!/bin/bash
if [ -f /tmp/log-last-check ]; then
  LASTSIZE="$(cat /tmp/log-last-check)"
  CURRENTSIZE="$(du -s /run/log/journal/*/system.journal)"
  if [ $(($CURRENTSIZE - $LASTSIZE)) -gt 250 ]; then
    echo $CURRENTSIZE > /tmp/log-last-check
    exit 1
  fi
else
  du -s /run/log/journal/*/system.journal | cut -f1 > /tmp/log-last-check
fi
exit 0
```

> A checagem de log não funciona quando o logdriver é baseado em **journald**, o arquivo para de crescer quando o journald atinge um tamanho limite.

## Rede

A checagem de rede utiliza das capacidades do bash em fazer chamadas TCP. Basicamente um pod qualquer é verificado, caso a chamada falhe as verificações se repetirão por 5 vezes, caso o problema persista o arquivo para que o **kured** reinicie o node será criado.

Se tudo der certo o comando retornará **0** e se o conteúdo de `/kured/reboot-required` for `check_timeout` o arquivo será removido, isso evitará a remoção do arquivo criado por outros problemas. Caso algum problema ocorra o retorno é **diferente de 0** e então começa a emitir o sinal para o **node-problem-detector**.

Atençaõ ao *timeout*, o comando leva em torno de 30 segundos para desistir.

```bash
#!/bin/bash
#exec 3<>/dev/tcp/10.96.30.1/443 # dns
exec 3<>/dev/tcp/10.244.166.139/80 # um pod qualquer
if [ $? -ne 0 ]; then
    COUNT=$(cat /tmp/timeout-last-check)
    COUNT=$(($COUNT + 1))
    echo $COUNT > /tmp/timeout-last-check
    if [ $COUNT -ge 5 ]; then
        echo 'check_timeout' > /kured/reboot-required # /var/run montado em /kured
        exit 1
    fi
else
    echo 0 > /tmp/timeout-last-check
    if [ "$(grep check_timeout /kured/reboot-required)" != '' ]; then
        rm /kured/reboot-required
    fi
fi
exit 0
```
