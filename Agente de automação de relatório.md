import os
import re
import time
from pypdf import PdfReader
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

print("--- 🤖 AGENTE DE AUDITORIA V8 INICIADO ---\n")

# --- ✍️ CONFIGURAÇÃO MANUAL ---
PASTA_DOWNLOADS = r"C:\Users\laura.alves\Downloads\Relatórios Externos Ráscal"
NOME_DO_ARQUIVO = "Rascal Itaim 29.02.24.pdf"

CAMINHO_PDF = os.path.join(PASTA_DOWNLOADS, NOME_DO_ARQUIVO)

# --- PASSO 1: VERIFICAÇÃO ---
if not os.path.exists(CAMINHO_PDF):
    print(f"❌ Erro: Arquivo '{NOME_DO_ARQUIVO}' não encontrado!")
    exit()

# --- PASSO 2: CONEXÃO COM O CHROME ---
try:
    chrome_options = Options()
    chrome_options.add_experimental_option("debuggerAddress", "127.0.0.1:9222")
    driver = webdriver.Chrome(options=chrome_options)
    print("   ✓ Conexão estabelecida com o Chrome!")
except Exception as e:
    print(f"\n❌ Erro de Conexão: {e}")
    print("👉 Certifique-se de fechar o Chrome normal e reabri-lo via CMD usando o comando:")
    print('   start chrome.exe --remote-debugging-port=9222 --user-data-dir="C:\\selenium\\AutomationProfile"')
    exit()

mapa_pdf = {}


def extrair_palavras_chave(texto):
    """Extrai palavras representativas de um texto para validação de segurança."""
    palavras = re.findall(r'[a-zA-ZÁ-ÿ]{4,}', texto.upper())
    ruidos = {"GESTT", "CHECKLIST", "CHECKLISTS", "RESPOSTA", "COMENTÁRIOS", "FOTOS", "TODOS", "ESTÃO", "PARA",
              "PÁGINA"}
    return [p for p in palavras if p not in ruidos]


try:
    # --- PASSO 3: LEITURA E MAPEAMENTO ISOLADO POR BLOCOS ---
    print("\n📌 Passo 3: Analisando o PDF e separando blocos exatos por ID...")
    leitor = PdfReader(CAMINHO_PDF)
    texto_completo = ""
    for pagina in leitor.pages:
        texto_completo += pagina.extract_text() + "\n"

    todos_ids = re.findall(r'\b(\d+\.\d+)\b', texto_completo)
    todos_ids = sorted(list(set(todos_ids)), key=lambda x: [int(num) for num in x.split('.')])

    for i, id_pergunta in enumerate(todos_ids):
        pos_atual = texto_completo.find(id_pergunta)
        if pos_atual == -1: continue

        if i + 1 < len(todos_ids):
            pos_proxima = texto_completo.find(todos_ids[i + 1], pos_atual + len(id_pergunta))
            if pos_proxima == -1:
                pos_proxima = pos_atual + 600
        else:
            pos_proxima = pos_atual + 600

        bloco_texto = texto_completo[pos_atual:pos_proxima]
        status = "NÃO OK" if "" in bloco_texto else "OK"

        comentario = ""
        if "Comentários" in bloco_texto:
            try:
                parte_comentario = bloco_texto.split("Comentários", 1)[1]
                corte = re.split(r'(?:Fotos|\b\d+\.\d+\b)', parte_comentario, flags=re.IGNORECASE)
                comentario = corte[0].strip(" :;-•\n\r")
            except Exception:
                pass

        palavras_chave = extrair_palavras_chave(bloco_texto.split("Resposta", 1)[0])

        mapa_pdf[id_pergunta] = {
            "status": status,
            "comentario": comentario,
            "palavras_chave": palavras_chave
        }

    print(f"   ✓ {len(mapa_pdf)} perguntas mapeadas com sucesso.")

    # --- PASSO 4: AUTOMAÇÃO WEB (PREENCHIMENTO DUPLO MATCH) ---
    print("\n📌 Passo 4: Pronto para preencher o site.")
    input("👉 Abra a página 1 no site e pressione [ENTER] aqui para iniciar...")

    for pagina_atual in range(1, 35):
        print(f"\n--- 📑 PROCESSANDO PÁGINA {pagina_atual} DO SITE ---")
        time.sleep(2.5)

        blocos = driver.find_elements(By.XPATH, "//cl-item | //div[contains(@class, 'cl-item')]")
        if not blocos:
            blocos = driver.find_elements(By.XPATH,
                                          "//cl-scale-evaluative/ancestor::div[contains(@class, 'card') or contains(@class, 'item') or @class='cl-item'][1]")

        for idx, bloco in enumerate(blocos, start=1):
            # 🚨 RESET COMPLETO DE VARIÁVEIS A CADA QUESTÃO DA TELA 🚨
            id_confirmado = None
            dados_match = None
            status_pdf = None
            comentario_pdf = ""

            try:
                texto_tela = (bloco.get_attribute("innerText") or bloco.get_attribute("textContent") or "").upper()
                match_id = re.search(r'(\d+)\s*\.\s*(\d+)', texto_tela)

                if match_id:
                    id_detectado = f"{match_id.group(1)}.{match_id.group(2)}"
                    if id_detectado in mapa_pdf:
                        dados_match = mapa_pdf[id_detectado]
                        id_confirmado = id_detectado

                if not id_confirmado:
                    palavras_tela = extrair_palavras_chave(texto_tela)
                    melhor_match_id = None
                    max_coincidencias = 0
                    for pdf_id, pdf_dados in mapa_pdf.items():
                        coincidencias = len(set(palavras_tela) & set(pdf_dados["palavras_chave"]))
                        if coincidencias > max_coincidencias:
                            max_coincidencias = coincidencias
                            melhor_match_id = pdf_id

                    if melhor_match_id and max_coincidencias >= 2:
                        id_confirmado = melhor_match_id
                        dados_match = mapa_pdf[melhor_match_id]

                # Só interage se houve um match real para esta pergunta da vez
                if id_confirmado and dados_match:
                    driver.execute_script("arguments[0].scrollIntoView({block: 'center', behavior: 'smooth'});", bloco)
                    time.sleep(1.0)

                    status_pdf = dados_match["status"]
                    comentario_pdf = dados_match["comentario"]

                    print(f"      📝 RESPONDENDO ITEM {id_confirmado} -> PDF diz: {status_pdf}")

                    xpath_feliz = ".//cl-scale-evaluative//cl-evaluative/div[contains(@class, 'smile')][2] | .//cl-scale-evaluative//div[2]/div/div | .//button[contains(@class, 'green')] | .//div[contains(@class, 'conforme')]"
                    xpath_triste = ".//cl-scale-evaluative//cl-evaluative/div[contains(@class, 'smile')][1] | .//cl-scale-evaluative//div[1]/div/div | .//button[contains(@class, 'red')] | .//div[contains(@class, 'nao-conforme')]"
                    xpath_comentar = ".//cl-addons-wrapper//button[contains(., 'Comentar')] | .//button[contains(., 'Comentar')] | .//span[contains(text(), 'Comentar')]/ancestor::button"
                    xpath_texto = ".//textarea | .//div[contains(@class, 'textarea')]/textarea"

                    if status_pdf == "OK":
                        btn_feliz = bloco.find_element(By.XPATH, xpath_feliz)
                        driver.execute_script("arguments[0].click();", btn_feliz)
                        time.sleep(0.6)
                    else:
                        btn_triste = bloco.find_element(By.XPATH, xpath_triste)
                        driver.execute_script("arguments[0].click();", btn_triste)
                        time.sleep(1.0)

                        btn_comentar = bloco.find_element(By.XPATH, xpath_comentar)
                        driver.execute_script("arguments[0].click();", btn_comentar)
                        time.sleep(0.8)

                        campo_texto = bloco.find_element(By.XPATH, xpath_texto)
                        campo_texto.clear()
                        # Se não houver comentário extraído, envia um texto padrão ou deixa em branco
                        if comentario_pdf:
                            campo_texto.send_keys(comentario_pdf)
                        else:
                            campo_texto.send_keys("Item não conforme conforme relatório.")
                        time.sleep(0.6)

            except Exception as e:
                if id_confirmado:
                    print(f"      ⚠️ Alerta no item {id_confirmado}: Falha ao interagir. Erro: {e}")
                pass

        try:
            xpath_prox = "//*[@id='checklist-nav-next'] | //cl-apply-navigation/div/button | //button[contains(., 'Próximo')] | //span[contains(text(), 'Próximo')]/ancestor::button"
            btn_prox = WebDriverWait(driver, 5).until(EC.presence_of_element_located((By.XPATH, xpath_prox)))
            driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
            time.sleep(0.5)
            if btn_prox.is_enabled():
                driver.execute_script("arguments[0].click();", btn_prox)
                time.sleep(3.0)
            else:
                break
        except Exception:
            break

    print("\n✅ AUDITORIA CONCLUÍDA COM SUCESSO!")

except Exception as e:
    print(f"❌ Erro Crítico: {e}")
