# DIO_Bootcamp_Azure_AI_102_Doc_Intelligence
Desafio - Document Intelligence - Validação de Cartão de Crédito


# AZURE 
## Recursos - Provisionados

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- Storage Blob

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- Document Intelligence

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img width="686" alt="Recursos Provisionados na Azure" src="https://github.com/user-attachments/assets/a58be121-d1cc-46fb-8a31-6ded5fab7f54">


# PYTHON 
## Estrutura:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img width="356" alt="Estrutura do Python" src="https://github.com/user-attachments/assets/42b7dfd1-3050-46f0-97f4-bf9bc4d20f8f">

## ➡️ requirements.py:

### Descrição:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Bibliotecas do Python utilizadas no Projeto.

### Código:
```
azure-core
azure-ai-documentintelligence
streamlit
azure-storage-blob
python-dotenv
```

## ➡️ .env:

### Descrição:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Variaveis de ambiente.

### Código:
```
RESOURCE-ENDPOINT = "https://<resource-xxxxxxxx>.cognitiveservices.azure.com/"
RESOURCE-KEY = "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
AZURE-STORAGE-CONNECTION-STRING = "DefaultEndpointsProtocol=https;FALSKDJFJASDFJAJSDFLJALSJDFLKJASLKDJFLKAJSDLKFJALKSDJFKAJSDKFK"
AZURE-STORAGE-CONTAINER-NAME = "cartoes"
```

## ➡️ utils > config.py:

### Descrição:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Recupera as variáveis de ambiente:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - Endpoint do Recurso;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - Key do Recurso;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - String de Conexão do Storage Blob;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - Nome do Storage Blob;

### Código:
```
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    ENDPOINT = os.getenv("RESOURCE-ENDPOINT")
    KEY = os.getenv("RESOURCE-KEY")
    STORAGE_CONNECTION_STRING = os.getenv("AZURE-STORAGE-CONNECTION-STRING")
    STORAGE_CONTAINER_NAME = os.getenv("AZURE-STORAGE-CONTAINER-NAME")
```

## ➡️ services > blob_service.py:

### Descrição:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Armazena no **Storage Blob** a Imagem que foi realizado o Upload.

### Código:
```
import os
import streamlit as st
from utils.config import Config
from azure.storage.blob import BlobServiceClient


def upload_file(file, file_name):
    try:
        
        # Conectando ao Storage Blob
        blob_service_client = BlobServiceClient.from_connection_string(Config.STORAGE_CONNECTION_STRING)
        
        # Definindo o Container Blob
        blob_client = blob_service_client.get_container_client(Config.STORAGE_CONTAINER_NAME)
        
        # Upload do Arquivo para o Container Blob
        blob_client.upload_blob(file_name,file, overwrite=True)

        return blob_client.url

    except Exception as e:
        st.error(f"Erro no upload do arquivo para o Azure Blob Storage: {e}")
        return None

```

## ➡️ services > credit_card_service.py:

### Descrição:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Submete a Imagem armazenada no `Storage Blob` para AI `Document Intelligence` analisar se é um Cartão de Crédito identificando cada um dos campos:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - Nome do Banco;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - Nome do Cliente;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - Número do Cartão;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - Data de Validade;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - Data de Expiração;

### Código:
```
import streamlit as st
from azure.core.credentials import AzureKeyCredential
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeDocumentRequest
from utils.config import Config

def analize_credit_card(blob_url,file):

    try:

        card_url = blob_url + "/" + file

        # Setando a Credencial da Azure
        azure_credential = AzureKeyCredential(Config.KEY)

        # Instanciando o Resource Document Intelligence
        document_client = DocumentIntelligenceClient(Config.ENDPOINT, azure_credential)

        # Analise da Imagem (Blob Storage) para validar se é um Cartão de Crédito
        card_info = document_client.begin_analyze_document("prebuilt-creditCard", AnalyzeDocumentRequest(url_source=card_url))
        
        # Resultado da Análise
        result = card_info.result()
        
        for document in result.documents:
            fields = document.get('fields',{})

            return {
                "bank_name": fields.get('IssuingBank',{}).get('content'), 
                "client_name": fields.get('CardHolderName',{}).get('content'),
                "card_number": fields.get('CardNumber',{}).get('content'),
                "expiry_date": fields.get('ExpirationDate',{}).get('content'),
                "valid_date": fields.get('ValidDate',{}).get('content')   
            }
        
    except Exception as ex:
        st.error(f"ERRO: Problema na validação do Cartão. [{ex}]")
        return None
```

## ➡️ app.py:

### Descrição:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
App Python, orquestração do frontend e backend (escolha do arquivo de imagem, armazenamento da imagem, validacao cartão, etc).

### Código:
```
import streamlit as st
from services.blob_service import upload_file
from services.credit_card_service import analize_credit_card

def configure_interface():
    st.title("[Azure - Document Intelligence]")

    #st.markdown(f"<h1 style='color: blue'>[Azure - Document Intelligence]</h1>", unsafe_allow_html=True)
    st.markdown(f"<h2 style='color: darkblue'>Validação Cartão de Crédito</h2>", unsafe_allow_html=True)

    st.markdown(f"<hr>", unsafe_allow_html=True)
    st.markdown(f"<h3> 1.0 - Escolha uma Imagem de Cartão de Crédito</h3>", unsafe_allow_html=True)
    uploaded_file = st.file_uploader("Escolha um arquivo", type=["png","jpg","jpeg"])
    st.markdown(f"<hr>", unsafe_allow_html=True)

    st.markdown(f"<h3> 2.0 - Processamento:</h3>", unsafe_allow_html=True)
    #st.markdown(f"<hr>", unsafe_allow_html=True)

    if uploaded_file is not None:
        #st.image(uploaded_file)

        st.markdown(f"<h4> 2.1 - Escolha do Arquivo: <b><i>Sucesso!!!</i></b></h4>", unsafe_allow_html=True)

        file_name = uploaded_file.name
        
        # Sobe para o Blob Storage
        blob_url = upload_file(uploaded_file, file_name)

        if blob_url:
            #st.write(f"02 - Upload do Arquivo {file_name} realizado com sucesso no Azure Blob Storage")
            #st.markdown(f"<hr>", unsafe_allow_html=True)
            st.markdown(f"<h4> 2.2 - Upload do Arquivo p/ Azure Blob Storage: <b><i>Sucesso!!!</b.</i></h4>", unsafe_allow_html=True)
            # Lê os dados da Imagem do Cartão
            credit_card_info = analize_credit_card(blob_url,file_name)
            
            show_image_and_validation(blob_url,file_name, credit_card_info)
        else:
            st.markdown(f"<h4> 2.2 - Upload do Arquivo p/ Azure Blob Storage: <b><i>ERRO!!!</b.</i></h4>", unsafe_allow_html=True)
    else:
        st.markdown(f"<h4> 2.1 - Escolha do Arquivo: <b><i>ERRO!!!</i></b></h4>", unsafe_allow_html=True)

def show_image_and_validation(url,arquivo, credit_card_info):
    
    arq_url = url + "/" + arquivo
    #st.markdown(f"<hr>", unsafe_allow_html=True)
    st.markdown(f"<h4> 2.3- Validação do Cartão:</h4>", unsafe_allow_html=True)
    st.image(arq_url, caption="Cartão para Validação",use_container_width=True)

    st.markdown(f"<h5> 2.3.1 - Resultado:</h5>", unsafe_allow_html=True)

    if credit_card_info and credit_card_info["client_name"]:
  
        st.markdown(f"<h2 style='color: green'>Cartão Válido</h1>", unsafe_allow_html=True)
        st.markdown(f"<h5> 2.3.2- Dados:</h5>", unsafe_allow_html=True)

        st.write(f"Banco Emissor: {credit_card_info['bank_name']}")
        st.write(f"Número do Cartão: {credit_card_info['card_number']}")
        st.write(f"Nome do Titular: {credit_card_info['client_name']}")
        st.write(f"Data de Expiração: {credit_card_info['expiry_date']}")
        st.write(f"Data de Validade: {credit_card_info['valid_date']}")

        st.markdown(f"<hr>", unsafe_allow_html=True)
    else:
        st.markdown(f"<h2 style='color: red'>Cartão Inválido</h1>", unsafe_allow_html=True)
        st.write(f"Este não é um cartão de crédito válido.")

if __name__ == "__main__":
    configure_interface()

```



 
