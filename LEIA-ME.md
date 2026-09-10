# Bot Mercado Livre → Discord

Código preparado para sua aplicação e seu serviço existente no Railway. Ainda precisa ser publicado e autorizado com sua conta. Nenhuma credencial está incluída nos arquivos.

## O que esta versão faz

- `/conectar`: link privado de autorização com PKCE e state, válido por 10 minutos.
- `/status`: situação da conexão salva (não é um teste ao vivo da API).
- `/relatorio dias:1`: Excel dos anúncios ativos e pedidos criados no período; aceita 1 a 30 dias. Resposta visível somente ao solicitante.
- A cada virada da hora: resumo da hora anterior, acumulado do dia e Excel no canal configurado. Horários apresentados em Brasília. Envia também se não houver pedidos.
- Renova e persiste os tokens do Mercado Livre. Comandos restritos ao seu ID do Discord, dentro do servidor configurado.
- Não altera anúncios, não envia mensagens a compradores e não realiza vendas ou movimentações financeiras.

As abas incluem anúncios, variações, pedidos, itens vendidos, pagamentos e notas. Preço, estoque, quantidade vendida acumulada, taxas e pagamentos são os valores devolvidos pela API, quando disponíveis. Valores ausentes ficam vazios. A listagem percorre páginas; falhas conhecidas interrompem o relatório, em vez de afirmar que uma lista parcial está completa.

## 1. Termine as variáveis no Railway

No serviço **gregarious-tranquility → Variables**, confira:

| Variável | Valor |
|---|---|
| `ML_CLIENT_ID` | `8210845058019559` |
| `ML_CLIENT_SECRET` | Seu segredo do Mercado Livre, já salvo |
| `ML_REDIRECT_URI` | `https://gregarious-tranquility-production-56e3.up.railway.app/oauth/callback` |
| `DISCORD_TOKEN` | Token do bot do Discord, já salvo |
| `DISCORD_CHANNEL_ID` | ID do canal privado de relatórios |
| `DISCORD_OWNER_ID` | **Seu ID de usuário no Discord**, não o ID do bot |
| `DATA_DIR` | `/data` |
| `PORT` | `8080` |

Para copiar seu ID: com o Modo desenvolvedor ativo no Discord, clique com o botão direito no seu próprio perfil e escolha **Copiar ID do usuário**. Use esse número em `DISCORD_OWNER_ID`.

O App ID do Discord é diferente do App ID do Mercado Livre. Este código não precisa de uma variável para o App ID do Discord.

## 2. Adicione um volume persistente

Antes de autorizar a conta, crie um **Volume** no projeto Railway e conecte-o ao serviço **gregarious-tranquility**, com **Mount Path `/data`**.

Você pode abrir o menu clicando com o botão direito na área do projeto ou pela paleta de comandos e procurar **Create Volume**. Selecione o serviço e configure o caminho de montagem. Aplique a mudança.

O volume mantém o arquivo `bot.sqlite3` com os tokens e o último horário enviado. Sem ele, um novo deploy pode perder a autorização. Use **uma única réplica** deste serviço. O banco contém segredos: não baixe nem publique seu conteúdo. Apenas o código é enviado ao GitHub.

Referência: [Volumes do Railway](https://docs.railway.com/volumes).

## 3. Coloque os arquivos no GitHub

1. Extraia o ZIP no computador.
2. Crie um repositório **privado** chamado `mercadolivre-discord` no GitHub.
3. Use **uploading an existing file** ou **Add file → Upload files**.
4. Envie os arquivos de dentro da pasta. `main.py`, `requirements.txt` e `Dockerfile` devem aparecer diretamente na raiz do repositório, sem uma pasta externa.
5. Salve com **Commit changes**. Inclua os arquivos de exclusão `.gitignore` e `.dockerignore`, se disponíveis no seletor. Eles não contêm segredos.

Nunca envie tokens, Client Secret, arquivos `.env` ou o banco ao repositório.

## 4. Conecte o serviço existente ao repositório

No Railway, abra **gregarious-tranquility → Settings → Source → Connect Repo**. Selecione o repositório novo e a branch principal. Se necessário, autorize o Railway a acessar esse repositório privado.

O `Dockerfile` instala as dependências e inicia `python main.py`. Não é necessário cadastrar um cron externo. Deixe a Root Directory em `/`.

Em **Networking**, confira se o domínio existente aponta para a porta **8080**, igual a `PORT`. Se configurar healthcheck, use `/health`; ele verifica o servidor HTTP, não a autorização do Mercado Livre.

Mantenha o serviço em execução contínua, com uma réplica e sem suspensão automática. Aplique o deploy. Nos logs, procure **Bot iniciado; comandos disponíveis no servidor configurado**. O uso continua sujeito ao saldo/plano do Railway.

## 5. Autorize pelo Discord

1. Entre no servidor que contém o canal configurado.
2. Digite `/conectar` e escolha o comando deste bot.
3. Clique no botão **Autorizar Mercado Livre**, visível somente para você.
4. Entre com a conta que possui os anúncios e autorize.
5. A página de retorno deve dizer **Mercado Livre conectado!**.
6. Volte ao Discord e execute `/relatorio dias:1`.

O endereço de retorno cadastrado no Mercado Livre deve ser idêntico ao da variável `ML_REDIRECT_URI`, inclusive protocolo e caminho, sem barra extra no final. Authorization Code, Refresh Token e PKCE devem estar habilitados. Permissões de leitura para anúncios, vendas e demais dados desejados.

O primeiro resumo automático será na próxima virada da hora após o bot iniciar, se já estiver autorizado. Exemplo: iniciou e autorizou às 14h20 → envia por volta de 15h, referente a 14h–15h. A consulta pode levar alguns minutos em contas grandes.

## Como interpretar os números

- O período é baseado na **data de criação do pedido**, com início inclusivo e fim exclusivo. Não é um relatório de recebimentos bancários.
- A planilha inclui todos os status. O resumo soma `total_amount` somente de pedidos atualmente `paid`, separados por moeda. Não mistura BRL e outras moedas.
- O total é **bruto**: não é lucro, valor líquido disponível ou conciliação de reembolsos e frete.
- O acumulado do dia segue o mesmo critério; a mensagem de meia-noite fecha o dia anterior.
- `sale_fee` é a taxa unitária informada no item do pedido, quando fornecida; não representa todas as cobranças.
- Dados de estoque podem variar conforme estoque compartilhado e modelo de anúncios. A versão atual mostra o valor da API sem reconciliar estoques de User Products.
- Não inclui descrições completas, visitas, publicidade, documentos fiscais, dados pessoais dos compradores, todas as tarifas, ou status detalhado de transportadoras. Portanto, esta é uma primeira versão de relatórios operacionais, não uma cópia integral de todas as telas da conta.
- Pedidos de anúncios atualmente pausados também são incluídos no período. A aba de anúncios contém apenas os atualmente ativos.
- Pagamentos e cancelamentos tardios não alteram mensagens antigas; consulte o período novamente com `/relatorio`.
- Não há recuperação automática de horas em que o serviço esteve desligado. Use `/relatorio` para consultar esse intervalo depois.
- O registro de envio reduz duplicações após reinícios. Uma queda entre a entrega e o registro local ainda pode deixar o estado de entrega incerto.
- Planilhas acima do limite de envio geram aviso, sem truncamento silencioso. Contas muito grandes podem exigir divisão de anexos e trabalhos de exportação em segundo plano.

## Se algo falhar

- **Comando não aparece:** confira o bot instalado com os escopos `bot` e `applications.commands`; reinicie o Discord. Os comandos são registrados apenas no servidor do canal configurado.
- **Comando restrito:** `DISCORD_OWNER_ID` precisa ser seu próprio ID e você deve estar no servidor configurado.
- **Bot offline:** confira o deploy, `DISCORD_TOKEN`, IDs e logs. Não é necessário ativar Message Content Intent.
- **Missing Access / Forbidden no Discord:** dê ao bot Ver canais, Enviar mensagens, Inserir links e Anexar arquivos no canal privado.
- **HTTP 403 do Mercado Livre:** confira permissões da aplicação e restrições da conta. Pode ser necessário reautorizar após modificar permissões.
- **Link expirado:** use `/conectar` de novo. Apenas o link mais recente é válido; não reutilize o endereço de retorno.
- **Falha ao renovar:** reautorize com `/conectar`. Revogação de acesso ou falha de rede durante rotação do refresh token pode exigir isso.
- **Página de retorno não abre:** confira deploy, porta 8080 e domínio. Nunca use o link público do domínio como substituto de `/conectar`.

## Validação e limites técnicos

Testes locais usam respostas simuladas para OAuth, paginação, cálculo monetário e geração do Excel. Nenhum teste acessa sua conta ou envia mensagens ao Discord. A validação real depende do primeiro deploy e `/relatorio` autorizado. A documentação do Mercado Livre retornou bloqueio de acesso na consulta desta sessão; os endpoints e campos utilizados precisam ser confirmados contra sua conta no teste integrado.

Para executar testes localmente: Python 3.12+, `python -m pip install -r requirements.txt`, depois `python -m unittest discover -s tests -v`.

Dependências permitem atualizações compatíveis dentro da versão principal; para builds estritamente reproduzíveis, fixe as versões após o teste integrado no seu ambiente.

Referências: [Discord — criação de bot](https://docs.discord.com/developers/quick-start/getting-started), [aiohttp — servidor HTTP](https://docs.aiohttp.org/en/stable/web_quickstart.html), [Mercado Livre — autenticação](https://developers.mercadolivre.com.br/pt_br/autenticacao-e-autorizacao).
