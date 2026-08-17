# processamento_de_boletos
Sistema de processamento de todos os tipo de boletos na AWS e integrando com o Zeev
Documentação da Arquitetura: Integração de Leitura de Boletos (Zeev & AWS)
1. Visão Geral da Arquitetura
O sistema tem como objetivo automatizar a leitura de dados de boletos bancários (linhas digitáveis, valores e vencimentos) anexados em processos do Zeev, extraindo as informações por meio de processamento de PDF e OCR (Amazon Textract), e devolvendo o JSON consolidado via API do Zeev.

Fluxo de Dados:

O processo no Zeev dispara uma requisição contendo os anexos dos boletos.

A mensageria enfileira a requisição para processamento assíncrono.

A função Lambda processa o arquivo, realiza o upload de backup no Amazon S3 e faz a varredura do PDF (com fallback automático para o Textract se necessário).

Os dados extraídos são salvos em formato JSON no S3 e enviados de volta via requisição HTTP (PATCH) para a API do Zeev.

Os logs e métricas de execução são monitorados via Amazon CloudWatch.

2. Componentes e Serviços AWS Utilizados
A. Amazon S3 (Armazenamento)
Nome do Bucket: zeev-boletos-producao

Finalidade:

Armazenar cópias de segurança de todos os arquivos PDF de boletos processados (/boletos/).

Armazenar os arquivos de saída consolidados com os dados extraídos (/json-saida/).

B. Amazon SQS (Fila de Mensageria)
Finalidade: Receber e armazenar de forma assíncrona as cargas úteis (payloads) enviadas pelo Zeev, garantindo que o processamento das Lambdas ocorra de maneira controlada e sem perda de dados em picos de requisições.

C. AWS Lambda (Computação Serverless)
Foram estruturadas funções em Python 3.15 (ou versão equivalente configurada no ambiente) utilizando bibliotecas especializadas para manipulação de PDF e requisições HTTP (pypdf, boto3, etc.).

Detalhamento das Funções/Módulos da Lambda:
lambda_handler(event, context) (Ponto de Entrada):

Processa os registros vindos da fila SQS.

Identifica o ID da instância do processo (processInstanceId) e mapeia a lista de boletos/anexos enviados no payload.

Executa o ciclo de download do anexo utilizando um token de autorização Bearer (ZEEV_TOKEN).

extrair_dados_boleto_de_pdf(caminho_local, file_bytes):

Estratégia 1 (PyPDF): Realiza a leitura nativa da primeira página do PDF. Aplica expressões regulares estruturadas e flexíveis para capturar códigos de barras que possuam separadores por ponto ou vírgula.

Estratégia 2 (Fallback Textract): Caso o PDF seja escaneado ou a leitura nativa não localize um código de barras válido de 47 dígitos pertencente a um banco suportado, aciona de forma inteligente o Amazon Textract (detect_document_text) estritamente na página 1.

decodificar_linha_digitavel(linha_dig):

Realiza a limpeza de caracteres especiais da linha digitável.

Extrai e formata matematicamente o valor do boleto (últimos 10 dígitos convertidos para centavos e formatados no padrão monetário brasileiro).

Extrai o fator de vencimento (posições 33 a 36) e calcula a data exata de vencimento considerando os padrões de bases de datas bancárias.

Envio de Retorno (Integração Zeev):

Monta o JSON formatado contendo o array de formValues (codigoDeBarrasBoleto, valor, vencimento) e executa uma requisição PATCH na API do Zeev ([https://dulub.zeev.it/api/2/formvalues/](https://dulub.zeev.it/api/2/formvalues/){process_id}).

D. Amazon Textract
Finalidade: Serviço de Inteligência Artificial/OCR da AWS acionado via código para extrair texto de PDFs escaneados ou imagens de boletos que não possuem camada de texto selecionável nativa.

E. IAM (Gerenciamento de Identidade e Acessos)
Finalidade: Conjunto de políticas e Roles de execução atribuídas à função Lambda, garantindo permissões mínimas necessárias para:

Gravar e ler objetos no bucket S3 (zeev-boletos-producao).

Consumir mensagens da fila SQS.

Executar chamadas de OCR no Amazon Textract (textract:DetectDocumentText).

Escrever logs de execução no CloudWatch.

F. Amazon CloudWatch
Finalidade:

Coleta automática de logs (print statements estruturados no código Python) para auditoria de download, rastreio de erros e depuração de layouts de boletos.

Monitoramento de métricas como tempo de execução, taxa de sucesso e falhas (errors/throttles).

G. AWS CloudShell
Finalidade: Ambiente de linha de comando baseado em navegador utilizado para interações rápidas com a AWS CLI, testes de pacotes, compactação de camadas (Lambda Layers) e gerenciamento direto dos recursos durante o desenvolvimento.

---

## 4. Como Manter Esta Documentação Atualizada?

Como este sistema envolve integrações externas (Zeev) e serviços em nuvem (AWS), siga estas regras simples sempre que houver mudanças:

### A. Mudanças no Código da Lambda
* **O que atualizar:** Se você alterar a lógica de leitura dos boletos, adicionar novos bancos na lista `BANCOS_VALIDOS` ou mudar regras de negócio, atualize a seção **C. AWS Lambda** descrevendo o que foi alterado.
* **Onde salvar:** Mantenha sempre a versão mais recente do código no repositório de controle de versão (como o GitHub ou GitLab).

### B. Mudanças na Infraestrutura da AWS
* **O que atualizar:** Se mudar o nome do Bucket S3, criar novas filas SQS ou alterar permissões de IAM, atualize os tópicos **S, SQS, S3 ou IAM** correspondentes.

### C. Alterações de Credenciais ou Tokens
* **O que atualizar:** Se o token de acesso do Zeev (`ZEEV_TOKEN`) expirar ou for renovado, atualize a nota de configuração nas Variáveis de Ambiente da Lambda. **Atenção:** Nunca coloque tokens de produção expostos publicamente na documentação.

### D. Frequência de Revisão
* Revise esta documentação sempre que houver uma **nova implantação em produção** ou quando o time do Zeev solicitar novos ajustes no layout dos boletos.
