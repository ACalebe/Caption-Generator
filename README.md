# Caption-Generator
	
import ttkbootstrap as ttk
import tkinter as tk
from tkinter import filedialog, messagebox
import yt_dlp
import os
import zipfile
import urllib.request
import subprocess
import shutil
import json
import whisper
import threading
from faster_whisper import WhisperModel
from plyer import notification

class YouTubeDownloader:
    def salvar_tema(self, tema):
        with open("config.json", "w") as f:
            json.dump({"tema": tema}, f)
    
    def validar_url_suportada(self, url):
        plataformas = ["youtube.com", "youtu.be", "instagram.com", "tiktok.com", "twitter.com", "x.com", "facebook.com"]
        return any(plataforma in url for plataforma in plataformas)
        if not self.validar_url_suportada(url):
            self.status_label.config(text="Plataforma ainda não suportada.", foreground="red")
            return

    def carregar_tema(self):
        if os.path.exists("config.json"):
            with open("config.json", "r") as f:
                try:
                    dados = json.load(f)
                    return dados.get("tema", "darkly")
                except:
                    return "darkly"
        return "darkly"
    
    def ao_trocar_aba(self, event):
        aba_index = event.widget.index("current")
        if aba_index == 0:  # Aba de vídeo
            self.verificar_clipboard(self.url_entry)
        elif aba_index == 1:  # Aba de corte
            self.verificar_clipboard(self.url_clip_entry)
        elif aba_index == 2:  # Aba de áudio
            self.verificar_clipboard(self.url_entry_audio)

    def verificar_clipboard(self, entry_widget):
        try:
            url = self.root.clipboard_get()
            if url.startswith("http") and any(site in url for site in ["youtube.com", "youtu.be", "instagram.com", "tiktok.com", "twitter.com", "x.com", "facebook.com"]):
                if not entry_widget.get().startswith("http"):
                    resposta = messagebox.askyesno(
                        "Detectado link!",
                        f"Detectamos um link na área de transferência:\n\n{url}\n\nDeseja colar no campo?"
                    )
                    if resposta:
                        entry_widget.delete(0, tk.END)
                        entry_widget.insert(0, url)
        except Exception:
            pass  # Ignora erros se não houver texto no clipboard ou se for binário

    def __init__(self, root):
        self.root = root
        self.root.title("Mídia Downloader (YouTube / Instagram / Áudio)")
        self.root.geometry("900x650")
        self.root.minsize(700, 500)
        self.root.resizable(True, True)

        self.transcription_cancel_event = threading.Event()

        self.use_faster_whisper = tk.BooleanVar(value=False)  # False = Whisper (modo padrão)

        self.video_info = {}
        self.available_formats = []
        self.pasta = os.getcwd()

        self.theme_var = tk.StringVar(value="darkly")
        self.create_widgets()

        tema_carregado = self.carregar_tema()
        self.theme_var.set(tema_carregado)
        ttk.Style().theme_use(tema_carregado)


    def centralizar(self, parent):
        frame = ttk.Frame(parent)
        frame.pack(expand=True)
        return frame
    
    def cancelar_transcricao(self):
        self.cancel_transcription = True
        self.status_label_transcribe.config(
            text="❌ Transcrição cancelada pelo usuário.",
            foreground="red"
        )


    def toggle_whisper_mode(self):
        self.use_faster_whisper.set(not self.use_faster_whisper.get())
        self.atualizar_toggle_whisper()

    def atualizar_toggle_whisper(self):
        if self.use_faster_whisper.get():
            self.toggle_label.config(text="🔁 Usando Faster-Whisper (mais rápido, menos preciso)")
            self.toggle_btn.config(text="Clique para usar Whisper (mais preciso)")
            if hasattr(self, 'model_label'):
                self.model_label.config(text="🔁 Modo: Faster-Whisper")
        else:
            self.toggle_label.config(text="🔄 Usando Whisper (mais preciso, mais lento)")
            self.toggle_btn.config(text="Clique para usar Faster-Whisper")
            if hasattr(self, 'model_label'):
                self.model_label.config(text="🔄 Modo: Whisper")

    def create_widgets(self):
        notebook = ttk.Notebook(self.root)
        self.video_tab = ttk.Frame(notebook)
        self.audio_tab = ttk.Frame(notebook)
        self.transcribe_tab = ttk.Frame(notebook)
        self.settings_tab = ttk.Frame(notebook)
        self.clip_tab = ttk.Frame(notebook)
        self.convert_audio_tab = ttk.Frame(notebook)

        notebook.add(self.convert_audio_tab, text="Converter Áudio")
        notebook.bind("<<NotebookTabChanged>>", self.ao_trocar_aba)
        notebook.add(self.video_tab, text="Baixar Vídeo")
        notebook.add(self.clip_tab, text="Corte de Vídeo")
        notebook.add(self.audio_tab, text="Baixar MP3")
        notebook.add(self.transcribe_tab, text="Gerar Legenda")
        notebook.add(self.settings_tab, text="Configurações")
        notebook.pack(expand=True, fill='both')

        # ------------------ Aba de vídeo ------------------ #
        video_frame = self.centralizar(self.video_tab)
        self.url_entry = ttk.Entry(video_frame, width=60)
        self.url_entry.pack(pady=10)
        self.url_entry.insert(0, "Cole a URL do vídeo")

        ttk.Button(video_frame, text="Buscar Vídeo", command=self.carregar_video).pack(pady=5)

        self.video_title = ttk.Label(video_frame, text="", wraplength=600, font=("Segoe UI", 12, "bold"))
        self.video_title.pack(pady=5)

        self.res_var = tk.StringVar(video_frame)
        self.res_menu = ttk.Combobox(video_frame, textvariable=self.res_var, state="readonly", width=70)
        self.res_menu.pack(pady=5)

        ttk.Button(video_frame, text="Escolher pasta", command=self.escolher_pasta).pack(pady=5)
        self.pasta_label = ttk.Label(video_frame, text=f"Pasta atual: {self.pasta}", wraplength=600)
        self.pasta_label.pack(pady=5)

        self.download_btn = ttk.Button(video_frame, text="Baixar Vídeo", command=self.baixar_video, state=tk.DISABLED)
        self.download_btn.pack(pady=10)

        self.status_label = ttk.Label(video_frame, text="", foreground="blue")
        self.status_label.pack(pady=10)

        # ------------------ Aba de conversão de áudio ------------------ #
        convert_audio_frame = self.centralizar(self.convert_audio_tab)

        ttk.Label(convert_audio_frame, text="Selecione o arquivo de áudio para converter:").pack(pady=10)

        self.select_audio_btn = ttk.Button(
            convert_audio_frame,
            text="Selecionar Arquivo",
            command=self.selecionar_arquivo_audio
        )
        self.select_audio_btn.pack(pady=5)

        self.selected_audio_label = ttk.Label(convert_audio_frame, text="Nenhum arquivo selecionado.")
        self.selected_audio_label.pack(pady=5)

        ttk.Button(convert_audio_frame, text="Escolher Pasta de Destino", command=self.escolher_pasta).pack(pady=5)
        self.output_pasta_label = ttk.Label(convert_audio_frame, text=f"Pasta atual: {self.pasta}", wraplength=600)
        self.output_pasta_label.pack(pady=5)

        self.convert_btn = ttk.Button(convert_audio_frame, text="Converter para MP3", command=self.converter_audio)
        self.convert_btn.pack(pady=10)

        self.status_label_convert = ttk.Label(convert_audio_frame, text="", foreground="blue")
        self.status_label_convert.pack(pady=10)


        # ------------------ Aba de áudio ------------------ #
        audio_frame = self.centralizar(self.audio_tab)
        self.url_entry_audio = ttk.Entry(audio_frame, width=60)
        self.url_entry_audio.pack(pady=10)
        self.url_entry_audio.insert(0, "Cole a URL do vídeo (YouTube ou Instagram)")

        ttk.Button(audio_frame, text="Escolher pasta", command=self.escolher_pasta).pack(pady=5)
        self.pasta_label_audio = ttk.Label(audio_frame, text=f"Pasta atual: {self.pasta}", wraplength=600)
        self.pasta_label_audio.pack(pady=5)

        self.audio_btn = ttk.Button(audio_frame, text="Baixar MP3", command=self.baixar_audio)
        self.audio_btn.pack(pady=10)

        self.status_label_audio = ttk.Label(audio_frame, text="", foreground="blue")
        self.status_label_audio.pack(pady=10)

        self.ffmpeg_btn = ttk.Button(audio_frame, text="Instalar FFmpeg automaticamente", command=self.instalar_ffmpeg)
        self.ffmpeg_btn.pack(pady=10)

        # ------------------ Aba de legenda ------------------ #
        transcribe_frame = self.centralizar(self.transcribe_tab)

        ttk.Label(transcribe_frame, text="Escolha um arquivo para gerar legenda:").pack(pady=10)

        self.transcribe_btn = ttk.Button(
            transcribe_frame,
            text="Selecionar arquivo",
            command=self.gerar_legenda_mp3
        )
        self.transcribe_btn.pack(pady=10)

        self.status_label_transcribe = ttk.Label(transcribe_frame, text="", foreground="blue")
        self.status_label_transcribe.pack(pady=10)

        self.progress_var_transcribe = tk.DoubleVar()
        self.progress_bar_transcribe = ttk.Progressbar(
            transcribe_frame,
            variable=self.progress_var_transcribe,
            maximum=100,
            length=400
        )
        self.progress_bar_transcribe.pack(pady=5)

        ttk.Label(transcribe_frame, text="Escolha o modelo do Whisper:").pack(pady=10)
        self.model_var = tk.StringVar(value="small")
        model_options = ["tiny", "base", "small", "medium", "large"]
        self.model_menu = ttk.Combobox(transcribe_frame, values=model_options, textvariable=self.model_var, state="readonly", width=30)
        self.model_menu.pack(pady=5)


        # Botão de cancelar transcrição (inicialmente oculto)
        self.cancel_transcription = False
        self.cancel_transcription_btn = ttk.Button(
            transcribe_frame,
            text="Cancelar Transcrição",
            command=self.cancelar_transcricao,
            bootstyle="danger"
        )
        self.cancel_transcription_btn.pack(pady=5)
        self.cancel_transcription_btn.pack_forget()

        # Botão de escolha entre Whisper e Faster-Whisper (toggle)
        self.use_faster_whisper = tk.BooleanVar(value=False)
        self.toggle_whisper_btn = ttk.Checkbutton(
            transcribe_frame,
            text="Usar Faster-Whisper (mais rápido, menos preciso)",
            variable=self.use_faster_whisper,
            bootstyle="success"
        )
        self.toggle_whisper_btn.pack(pady=5)



        # ------------------ Aba de configurações ------------------ #
        settings_frame = self.centralizar(self.settings_tab)
        ttk.Label(settings_frame, text="Escolha um tema para o aplicativo:").pack(pady=10)
        theme_options = ["darkly", "cosmo", "litera", "flatly", "vapor", "morph"]
        ttk.Combobox(settings_frame, values=theme_options, textvariable=self.theme_var, state="readonly", width=30).pack(pady=5)
        ttk.Button(settings_frame, text="Aplicar Tema", command=self.aplicar_tema).pack(pady=10)

        # Toggle de whisper
        self.toggle_label = ttk.Label(settings_frame, text="", font=("Segoe UI", 10))
        self.toggle_label.pack(pady=5)

        self.toggle_btn = ttk.Button(settings_frame, command=self.toggle_whisper_mode)
        self.toggle_btn.pack()

        self.atualizar_toggle_whisper()

        self.verificar_ffmpeg()
        self.criar_aba_corte()

                # No final da aba de Vídeo
        self.progress_var_video = tk.DoubleVar()
        self.progress_bar_video = ttk.Progressbar(
            video_frame,
            variable=self.progress_var_video,
            maximum=100,
            length=400
        )
        self.progress_bar_video.pack(pady=5)

        # No final da aba de Áudio
        self.progress_var_audio = tk.DoubleVar()
        self.progress_bar_audio = ttk.Progressbar(
            audio_frame,
            variable=self.progress_var_audio,
            maximum=100,
            length=400
        )
        self.progress_bar_audio.pack(pady=5)




    def aplicar_tema(self):
        novo_tema = self.theme_var.get()
        try:
            ttk.Style().theme_use(novo_tema)
            self.salvar_tema(novo_tema)
            self.status_label.config(text=f"Tema aplicado: {novo_tema}", foreground="green")
        except Exception as e:
            messagebox.showerror("Erro ao aplicar tema", f"Não foi possível aplicar o tema.\n{e}")


    def verificar_ffmpeg(self):
        if shutil.which("ffmpeg"):
            self.ffmpeg_btn.pack_forget()
        else:
            self.url_entry_audio.pack_forget()
            self.audio_btn.pack_forget()
            self.pasta_label_audio.pack_forget()

    def carregar_video(self):
        url = self.url_entry.get().strip()
        if not url.startswith("http"):
            self.status_label.config(text="URL inválida.", foreground="red")
            return

        try:
            ydl_opts = {'quiet': True, 'skip_download': True, 'forcejson': True}
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                info = ydl.extract_info(url, download=False)
                self.video_info = info
                self.video_title.config(text=info.get("title", "Título não encontrado"))

                formats = info.get("formats", [])
                formats = [f for f in formats if f.get("vcodec") != "none"]
                formats.sort(key=lambda x: x.get("height", 0), reverse=True)

                self.available_formats = []
                res_list = []
                for f in formats:
                    resolution = f.get("height") or "???"
                    ext = f.get("ext", "")
                    has_audio = f.get("acodec") != "none"
                    audio_icon = "🔈" if has_audio else "🔇"
                    label = "Alta qualidade" if resolution != "???" and int(resolution) >= 720 else (
                            "Baixa qualidade" if resolution != "???" and int(resolution) <= 360 else "Média qualidade")
                    filesize = f.get("filesize") or f.get("filesize_approx")
                    size_str = f"{round(filesize / (1024 * 1024), 1)}MB" if filesize else "Tamanho desconhecido"
                    desc = f"{audio_icon} {resolution}p • {ext.upper()} • {label} • {size_str}"
                    self.available_formats.append(f)
                    res_list.append(desc)

                self.res_menu["values"] = res_list
                self.res_menu.current(0 if res_list else -1)
                self.download_btn.config(state=(tk.NORMAL if res_list else tk.DISABLED))
                self.status_label.config(text="Vídeo carregado com sucesso!", foreground="green")

        except Exception as e:
            self.status_label.config(text=f"Erro: {e}", foreground="red")

    def escolher_pasta(self):
        pasta = filedialog.askdirectory()
        if pasta:
            self.pasta = pasta
            self.pasta_label.config(text=f"Pasta atual: {self.pasta}")
            self.pasta_label_audio.config(text=f"Pasta atual: {self.pasta}")
            self.output_pasta_label.config(text=f"Pasta atual: {self.pasta}")

    def baixar_video(self):
        url = self.url_entry.get().strip()
        if not url.startswith("http"):
            self.status_label.config(text="URL inválida.", foreground="red")
            return

        selection = self.res_var.get()
        if not selection:
            self.status_label.config(text="Selecione uma resolução.", foreground="red")
            return

        index = self.res_menu.current()
        selected_format = self.available_formats[index]
        format_id = selected_format.get("format_id")

        ydl_opts = {
            'format': f"{format_id}+bestaudio/best",
            'outtmpl': os.path.join(self.pasta, '%(title)s.%(ext)s'),
            'quiet': True,
            'merge_output_format': 'mp4',
            'postprocessors': [{'key': 'FFmpegVideoConvertor', 'preferedformat': 'mp4'}],
            'progress_hooks': [self.progresso_download],
        }

        self.status_label.config(text="Iniciando download com áudio embutido...", foreground="blue")
        self.root.update_idletasks()

        try:
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                ydl.download([url])
            self.status_label.config(text="✅ Vídeo com áudio baixado com sucesso!", foreground="green")
            notification.notify(
                title="Download concluído",
                message="O vídeo foi baixado com sucesso!",
                app_name="Mídia Downloader"
            )
        except Exception as e:
            self.status_label.config(text=f"Erro no download: {e}", foreground="red")


    def baixar_audio(self):
        url = self.url_entry_audio.get().strip()
        if not url.startswith("http"):
            self.status_label_audio.config(text="URL inválida.", foreground="red")
            return

        ydl_opts = {
            'format': 'bestaudio/best',
            'outtmpl': os.path.join(self.pasta, '%(title)s.%(ext)s'),
            'quiet': True,
            'postprocessors': [{'key': 'FFmpegExtractAudio', 'preferredcodec': 'mp3', 'preferredquality': '192'}],
            'progress_hooks': [self.progresso_download_audio],
        }

        self.status_label_audio.config(text="Iniciando download de áudio...", foreground="blue")
        self.root.update_idletasks()

        try:
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                ydl.download([url])
            self.status_label_audio.config(text="✅ Áudio baixado com sucesso!", foreground="green")
            notification.notify(
                title="Download concluído",
                message="O áudio foi baixado com sucesso!",
                app_name="Mídia Downloader"
            )
        except Exception as e:
            self.status_label_audio.config(text=f"Erro no download: {e}", foreground="red")

    def gerar_legenda_mp3(self):
        import os

        file_path = filedialog.askopenfilename(
            filetypes=[
                ("Arquivos de Mídia", "*.mp3 *.mp4 *.wav *.aac *.flac *.m4a *.mov *.mkv *.avi *.webm"),
                ("Todos os arquivos", "*.*")
            ]
        )
        if not file_path:
            return

        print(f"[DEBUG] file_path: {file_path}")

        self.cancel_transcription = False
        self.cancel_transcription_btn.pack()
        self.progress_var_transcribe.set(0)
        self.status_label_transcribe.config(text="Iniciando transcrição...", foreground="blue")
        self.root.update_idletasks()

        def format_time(seconds):
            hrs = int(seconds // 3600)
            mins = int((seconds % 3600) // 60)
            secs = int(seconds % 60)
            millis = int((seconds - int(seconds)) * 1000)
            return f"{hrs:02}:{mins:02}:{secs:02},{millis:03}"

        try:
            # Escolha do modelo
            if self.use_faster_whisper.get():
                from faster_whisper import WhisperModel
                modelo_escolhido = self.model_var.get()
                model = WhisperModel(modelo_escolhido, compute_type="int8")
                segment_generator, _ = model.transcribe(file_path, beam_size=5, language="pt")
                segments = list(segment_generator)
            else:
                import whisper
                modelo_escolhido = self.model_var.get()
                model = whisper.load_model(modelo_escolhido)
                result = model.transcribe(file_path, language="pt")
                segments = [
                    type("Segment", (), {"start": seg["start"], "end": seg["end"], "text": seg["text"]})
                    for seg in result["segments"]
                ]

            total = len(segments)
            srt_path = os.path.splitext(file_path)[0] + ".srt"

            with open(srt_path, "w", encoding="utf-8") as f:
                for i, segment in enumerate(segments, start=1):
                    if self.cancel_transcription:
                        self.status_label_transcribe.config(text="❌ Transcrição cancelada pelo usuário.", foreground="red")
                        return

                    f.write(f"{i}\n")
                    f.write(f"{format_time(segment.start)} --> {format_time(segment.end)}\n")
                    f.write(segment.text.strip() + "\n\n")

                    self.progress_var_transcribe.set((i / total) * 100)
                    self.root.update_idletasks()

            self.status_label_transcribe.config(text="✅ Legenda gerada com sucesso!", foreground="green")
            notification.notify(
                title="Transcrição concluída",
                message="A legenda foi gerada com sucesso!",
                app_name="Mídia Downloader"
            )

            # 🟢 Aqui é a mudança importante
            try:
                pasta_origem = os.path.dirname(file_path)
                print(f"[DEBUG] pasta_origem: {pasta_origem}")
                os.startfile(pasta_origem)  # <- isso sempre funciona mesmo com acentos
            except Exception as e:
                print(f"Erro ao abrir a pasta: {e}")

        except Exception as e:
            self.status_label_transcribe.config(text=f"Erro ao gerar legenda: {e}", foreground="red")
        finally:
            self.progress_var_transcribe.set(0)
            self.cancel_transcription_btn.pack_forget()









    def progresso_download(self, d):
        if d['status'] == 'downloading':
            percent_str = d.get('_percent_str', '').strip().replace('%', '')
            try:
                percent = float(percent_str)
            except ValueError:
                percent = 0.0
            self.status_label.config(text=f"Baixando... {percent:.1f}%", foreground="blue")
            self.progress_var_video.set(percent)
            self.root.update_idletasks()
        elif d['status'] == 'finished':
            self.progress_var_video.set(100)


    def progresso_download_audio(self, d):
        if d['status'] == 'downloading':
            percent_str = d.get('_percent_str', '').strip().replace('%', '')
            try:
                percent = float(percent_str)
            except ValueError:
                percent = 0.0
            self.status_label_audio.config(text=f"Baixando... {percent:.1f}%", foreground="blue")
            self.progress_var_audio.set(percent)
            self.root.update_idletasks()
        elif d['status'] == 'finished':
            self.progress_var_audio.set(100)


    def instalar_ffmpeg(self):
        try:
            url = "https://www.gyan.dev/ffmpeg/builds/ffmpeg-release-essentials.zip"
            pasta_destino = os.path.join(os.environ["USERPROFILE"], "ffmpeg_auto")
            zip_path = os.path.join(pasta_destino, "ffmpeg.zip")
            os.makedirs(pasta_destino, exist_ok=True)

            self.status_label_audio.config(text="Baixando FFmpeg...", foreground="blue")
            self.root.update_idletasks()
            urllib.request.urlretrieve(url, zip_path)

            self.status_label_audio.config(text="Extraindo FFmpeg...", foreground="blue")
            with zipfile.ZipFile(zip_path, 'r') as zip_ref:
                zip_ref.extractall(pasta_destino)

            ffmpeg_dir = next((os.path.join(pasta_destino, d, "bin")
                               for d in os.listdir(pasta_destino)
                               if os.path.isdir(os.path.join(pasta_destino, d)) and "ffmpeg" in d.lower()), None)

            if not ffmpeg_dir:
                self.status_label_audio.config(text="Erro ao localizar a pasta bin do FFmpeg.", foreground="red")
                return

            subprocess.run(f'setx PATH \"%PATH%;{ffmpeg_dir}\"', shell=True)
            self.status_label_audio.config(text="✅ FFmpeg instalado! Reinicie o computador para aplicar.", foreground="green")
            self.ffmpeg_btn.pack_forget()
            self.url_entry_audio.pack(pady=10)
            self.audio_btn.pack(pady=10)
            self.pasta_label_audio.pack(pady=5)
        except Exception as e:
            self.status_label_audio.config(text=f"Erro na instalação: {e}", foreground="red")

    def criar_aba_corte(self):
        frame = ttk.Frame(self.clip_tab)
        frame.pack(fill="both", expand=True, pady=20)

        ttk.Label(frame, text="Cole a URL do vídeo (YouTube)").pack(pady=5)
        self.url_clip_entry = ttk.Entry(frame, width=60)
        self.url_clip_entry.pack(pady=5)

        # Início do corte
        ttk.Label(frame, text="Início do corte").pack(pady=(10, 2))
        inicio_frame = ttk.Frame(frame)
        inicio_frame.pack(pady=2)
        ttk.Label(inicio_frame, text="Min:").pack(side="left")
        self.start_min_entry = ttk.Entry(inicio_frame, width=5)
        self.start_min_entry.pack(side="left", padx=(0, 10))
        ttk.Label(inicio_frame, text="Seg:").pack(side="left")
        self.start_sec_entry = ttk.Entry(inicio_frame, width=5)
        self.start_sec_entry.pack(side="left")

        # Fim do corte
        ttk.Label(frame, text="Fim do corte").pack(pady=(10, 2))
        fim_frame = ttk.Frame(frame)
        fim_frame.pack(pady=2)
        ttk.Label(fim_frame, text="Min:").pack(side="left")
        self.end_min_entry = ttk.Entry(fim_frame, width=5)
        self.end_min_entry.pack(side="left", padx=(0, 10))
        ttk.Label(fim_frame, text="Seg:").pack(side="left")
        self.end_sec_entry = ttk.Entry(fim_frame, width=5)
        self.end_sec_entry.pack(side="left")

        # Escolher pasta e botão
        ttk.Button(frame, text="Escolher Pasta", command=self.escolher_pasta).pack(pady=5)
        self.clip_pasta_label = ttk.Label(frame, text=f"Pasta atual: {self.pasta}")
        self.clip_pasta_label.pack(pady=5)

        self.clip_status = ttk.Label(frame, text="", foreground="blue")
        self.clip_status.pack(pady=5)

        ttk.Button(frame, text="Cortar e Baixar", command=self.cortar_trecho_video).pack(pady=10)

    def selecionar_arquivo_audio(self):
        file_path = filedialog.askopenfilename(
            filetypes=[
                ("Arquivos de Áudio", "*.opus *.wav *.ogg *.aac *.m4a *.flac *.mp3"),
                ("Todos os arquivos", "*.*")
            ]
        )
        if file_path:
            self.selected_audio_file = file_path
            self.selected_audio_label.config(text=f"Arquivo selecionado: {os.path.basename(file_path)}")
        else:
            self.selected_audio_label.config(text="Nenhum arquivo selecionado.")

    def converter_audio(self):
        if not hasattr(self, 'selected_audio_file') or not self.selected_audio_file:
            self.status_label_convert.config(text="Nenhum arquivo selecionado.", foreground="red")
            return

        input_file = self.selected_audio_file
        output_file = os.path.splitext(os.path.basename(input_file))[0] + ".mp3"
        output_path = os.path.join(self.pasta, output_file)

        self.status_label_convert.config(text="Iniciando conversão...", foreground="blue")
        self.root.update_idletasks()

        try:
            comando = [
                "ffmpeg", "-i", input_file, "-vn", "-acodec", "libmp3lame", "-ab", "192k", output_path
            ]
            subprocess.run(comando, check=True)
            self.status_label_convert.config(text=f"✅ Conversão concluída: {output_file}", foreground="green")
            notification.notify(
                title="Conversão concluída",
                message=f"O arquivo foi convertido para MP3: {output_file}",
                app_name="Mídia Downloader"
            )
        except Exception as e:
            self.status_label_convert.config(text=f"Erro na conversão: {e}", foreground="red")

    def cortar_trecho_video(self):
        url = self.url_clip_entry.get().strip()

        try:
            start_min = int(self.start_min_entry.get() or 0)
            start_sec = int(self.start_sec_entry.get() or 0)
            end_min = int(self.end_min_entry.get() or 0)
            end_sec = int(self.end_sec_entry.get() or 0)
        except ValueError:
            self.clip_status.config(text="Preencha os campos de tempo apenas com números.", foreground="red")
            return

        start_total = start_min * 60 + start_sec
        end_total = end_min * 60 + end_sec

        if not url or end_total <= start_total:
            self.clip_status.config(text="Verifique a URL e os tempos preenchidos.", foreground="red")
            return

        # Converte para formato hh:mm:ss
        def format_time(seconds):
            h = seconds // 3600
            m = (seconds % 3600) // 60
            s = seconds % 60
            return f"{h:02}:{m:02}:{s:02}"

        inicio_str = format_time(start_total)
        fim_str = format_time(end_total)

        self.clip_status.config(text="Baixando somente o trecho desejado...", foreground="blue")
        self.root.update_idletasks()

        try:
            output_path = os.path.join(self.pasta, f"trecho_{inicio_str.replace(':', '-')}_a_{fim_str.replace(':', '-')}.mp4")

            ydl_opts = {
                'format': 'bestvideo+bestaudio/best',
                'download_sections': [f'*{inicio_str}-{fim_str}'],
                'outtmpl': output_path,
                'merge_output_format': 'mp4',
                'quiet': True,
                'postprocessors': [{'key': 'FFmpegVideoConvertor', 'preferedformat': 'mp4'}],
            }

            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                ydl.download([url])

            self.clip_status.config(text=f"✅ Trecho salvo: {output_path}", foreground="green")

        except Exception as e:
            self.clip_status.config(text=f"Erro ao cortar: {e}", foreground="red")
    
if __name__ == "__main__":
    tema = "darkly"
    if os.path.exists("config.json"):
        with open("config.json", "r") as f:
            try:
                config = json.load(f)
                tema = config.get("tema", "darkly")
            except:
                tema = "darkly"
    root = ttk.Window(themename=tema)
    app = YouTubeDownloader(root)
    root.mainloop()
