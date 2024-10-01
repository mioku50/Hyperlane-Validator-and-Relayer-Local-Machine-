# Ретранслятора.



**Validator (Валидаторы)**

Валидаторы Hyperlane — это агенты оффчейна, отвечающие за безопасность: они наблюдают за сообщениями в почтовом ящике цепочек и при необходимости подписывают корень merkle, подтверждающий текущее состояние почтового ящика.

Эта подпись хранится и делается общедоступной , которая затем используется ретранслятором вне цепочки и модулями межцепочечной безопасности в сети. Валидаторы не объединены в сеть и не нуждаются в достижении консенсуса; они также не осуществляют регулярных транзакций в ончейн.

Следуя этому руководству, вы сможете запустить валидатор Hyperlane на любой из существующих цепочек, на которых работает протокол. Валидаторы Hyperlane запускаются на основе цепочек-оригиналов.

В данном гайде мы запустим валидатора на цепочке Base Sepolia, но вы можете запустить на любой другой.

**Relayer (Ретрансляторы)**

Для доставки каждого сообщения Hyperlane требуется две транзакции: одна в цепочке отправителя для отправки сообщения, другая — в цепочке получателя для его получения. За отправку второй транзакции отвечает ретранслятор.

Релайнеры Hyperlane настраиваются для передачи сообщений между одной или несколькими цепочками происхождения и цепочками назначения. Relayer не имеет специальных разрешений в Hyperlane. Если ключи ретранслятора скомпрометированы, риску подвергаются только токены, хранящиеся на этих ключах.

**Установка Валидатора**

_**Минимальные технические характеристики**_

_**Подготовка сервера**_

```
sudo apt update
apt install curl iptables build-essential git wget jq make gcc nano tmux htop nvme-cli pkg-config libssl-dev libleveldb-dev libgmp3-dev tar clang bsdmainutils ncdu unzip llvm libudev-dev make protobuf-compiler -ysudo apt-get install git cargo clang cmake build-essential pkg-config openssl libssl-dev protobuf-compilercurl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

_**Ставим Foundry**_

```
curl -L https://foundry.paradigm.xyz | bash
source /root/.bashrc
foundryup
```

_**Ставим Hyperlane CLI**_

```
npm install -g @hyperlane-xyz/cli
```

_Если возникнут ошибки при установке и NPM будет ругаться, то устанавливаем NVM и далее снова ставим CLI_

```
curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
source ~/.nvm/nvm.sh
nvm install --lts
nvm use --lts
npm --version
```

Далее инициализируем конфигурацию multisig ISM

```
hyperlane core init --advanced
```

_Enter threshold of validators (number) for message ID multisig ISM —_ пишем 1

_Enter validator addresses (comma separated list) for message ID multisig ISM — вписываем ваш EVM Адрес_

_Select default hook type — выбираем merkleTreeHook_

_Далее выбираем — aggregationHook_

_Далее выбираем_ protocolFee _и на следующие вопросы отвечаем:_

_For protocol fee hook, enter owner address — Ваш EVM адрес_

_Use this same address — YES_

_Enter max protocol fee for protocol fee hook — 100000000000000000_

_Enter protocol fee for protocol fee hook — 0.1_

_**Делаем Deploy**_

_**В выбранной вашей сети должно быть как минимум 0.02 ETH**_

```
hyperlane core deploy
```

HYP\_Key — вставляем наш _**Приватный Ключ**_ EVM (Metamask)

_Select network type — выбираем testnet или mainnet_

_Select chain to connect — я выбрал basesepolia_

_Do you want to use an API key to verify on this (basesepolia) chain’s block explorer — N (No) и ENTER_

_Если все прошло хорошо то должно быть так_

_**Теперь мы готовы сгенерировать конфигурацию агента, используя наши развернутые контракты**_

```
hyperlane registry agent-config --chains basesepolia
```

_**Прописываем зависимости**_

```
export CONFIG_FILES=$HOME/configs/agent-config.json
mkdir r -p tmp/hyperlane-validator-signatures-basesepolia
export VALIDATOR_SIGNATURES_DIR=/tmp/hyperlane-validator-signatures-basesepolia
mkdir -p $VALIDATOR_SIGNATURES_DIR
```

**Запуск Валидатора**

_**Клонируем репозиторий**_

```
git clone git@github.com:hyperlane-xyz/hyperlane-monorepo.git
```

_От нас потребуют разрешения в виде SSH Key и пароль к нему_

_Если у вас нет SSH Ключа то сделаем его_

```
ssh-keygen -t rsa -b 4096 -C ваш Email
cat ~/.ssh/id_rsa.pub
```

Копируем ключ который выдала нам команда и отправляемся на свою страницу в Github. Нажимаем “Setting” в правом углу и попадаем в настройки профиля.

Далее нажимаем SSH and GPG Keys и в во вкладке SSH key вводим наше скопированное ранее значение.

_После успешного клонирования репозитория **прописываем следующее**_

```
screen -s hyp val
cd hyperlane-monorepo
cd rust
```

_**Далее запускаем Валидатор**_

```
cargo run --release --bin validator -- \
    --db ./hyperlane_db_validator_basesepolia \
    --originChainName basesepolia\
    --checkpointSyncer.type localStorage \
    --checkpointSyncer.path $VALIDATOR_SIGNATURES_DIR \
    --validator.key <your_validator_key
```

\<your\_validator\_key> _Ваш приватный ключ EVM (Metamask)_

_Ждем “Билда” от получаса до нескольких часов и по завершению должны пойти подобные логи:_

_Все! Валидатор успешно установлен_

**Запускаем Relayer**

```
screen -s hyp rel
cd hyperlane-monorepo
cd rust
```

```
cargo run --release --bin relayer -- \
    --db ./hyperlane_db_relayer \
    --relayChains <chain_1_name>,<chain_2_name> \
    --allowLocalCheckpointSyncers true \
    --defaultSigner.key <your_relayer_key> \
    --metrics-port 9091
```

\<chain\_1\_name>,\<chain\_2\_name> _например basesepolia,sepolia_

\<your\_relayer\_key> _ваш приватный ключ EVM (Metamask)_

_**Логи**_

```
screen -x hyp val
screen -x hyp rel
```

_**Отправка тестовых сообщений**_

_Вы можете проверить, что все работает правильно, отправив тестовое сообщение между парами цепочек. Запустите следующую команду с помощью CLI_

```
hyperlane send message --key $PRIVATE_KEY
```

_Официальная документация_ — [https://docs.hyperlane.xyz/docs/guides/deploy-hyperlane-local-agents](https://docs.hyperlane.xyz/docs/guides/deploy-hyperlane-local-agents)

_Twitter_ — [https://x.com/hyperlane](https://x.com/hyperlane)

_Discord_ — [https://discord.com/invite/hyperlane](https://discord.com/invite/hyperlane)
