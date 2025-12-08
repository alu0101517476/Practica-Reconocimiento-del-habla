# Practica-Reconocimiento-del-habla

Autor: Eric Bermúdez Hernández

Email: alu0101517476@ull.edu.es

---

# Objetivo de la práctica

El objetivo de la práctica es la implementación de la librería Whisper en Unity para el reconocimiento de voz en tiempo real. El objetivo final de esta implementación es controlar propiedades de objetos en la escena (en este caso, el color de un cubo) mediante comandos de voz hablados en español o en otros idiomas.

## Instalación del paquete

Para integrar Whisper en el proyecto, se utilizó la implementación para Unity desarrollada por el usuario `Macoron`, descargando el repositorio como un zip y después añadiendolo mediante el Unity Hub y el botón `Add` la carpeta obtenida después de descomprimir el .zip del repositorio. El enlace del repositorio es el siguiente: https://github.com/Macoron/whisper.unity/tree/master

## Configuración de Escenas y Modelos

Una vez importado el contenido del repositorio en una escena, se procedió a preparar el entorno de trabajo. Se seleccionó en la sección del paquete Whisper el sample `Microphone Demo` que sirve como base para la captura de audio en tiempo real.

## Implementación del Código

Se modificó el script de ejemplo MicrophoneDemo.cs para incluir la lógica de detección de la palabra clave `Rojo` y la referencia al objeto 3D que debe cambiar de color. El script final gestiona la grabación, la transcripción en tiempo real y la ejecución de comandos.
A continuación, se presenta el código completo del script MicrophoneDemo.cs utilizado:

```C#
using System.Diagnostics;
using UnityEngine;
using UnityEngine.UI;
using Whisper.Utils;
using Button = UnityEngine.UI.Button;
using Toggle = UnityEngine.UI.Toggle;

namespace Whisper.Samples
{
    /// <summary>
    /// Record audio clip from microphone and make a transcription.
    /// </summary>
    public class MicrophoneDemo : MonoBehaviour
    {
        public WhisperManager whisper;
        public MicrophoneRecord microphoneRecord;
        public bool streamSegments = true;
        public bool printLanguage = true;

        [Header("UI")] 
        public Button button;
        public Text buttonText;
        public Text outputText;
        public Text timeText;
        public Dropdown languageDropdown;
        public Toggle translateToggle;
        public Toggle vadToggle;
        public ScrollRect scroll;
        
        [Header("Configuración Color")]
        // --- NUEVA VARIABLE: Aquí arrastraremos el cubo ---
        public Renderer cuboRenderer; 

        private string _buffer;

        private void Awake()
        {
            whisper.OnNewSegment += OnNewSegment;
            whisper.OnProgress += OnProgressHandler;
            
            microphoneRecord.OnRecordStop += OnRecordStop;
            
            button.onClick.AddListener(OnButtonPressed);
            languageDropdown.value = languageDropdown.options
                .FindIndex(op => op.text == whisper.language);
            languageDropdown.onValueChanged.AddListener(OnLanguageChanged);

            translateToggle.isOn = whisper.translateToEnglish;
            translateToggle.onValueChanged.AddListener(OnTranslateChanged);

            vadToggle.isOn = microphoneRecord.vadStop;
            vadToggle.onValueChanged.AddListener(OnVadChanged);
        }

        private void OnVadChanged(bool vadStop)
        {
            microphoneRecord.vadStop = vadStop;
        }

        private void OnButtonPressed()
        {
            if (!microphoneRecord.IsRecording)
            {
                microphoneRecord.StartRecord();
                buttonText.text = "Stop";
            }
            else
            {
                microphoneRecord.StopRecord();
                buttonText.text = "Record";
            }
        }
        
        private async void OnRecordStop(AudioChunk recordedAudio)
        {
            buttonText.text = "Record";
            _buffer = "";

            var sw = new Stopwatch();
            sw.Start();
            
            var res = await whisper.GetTextAsync(recordedAudio.Data, recordedAudio.Frequency, recordedAudio.Channels);
            if (res == null || !outputText) 
                return;

            var time = sw.ElapsedMilliseconds;
            var rate = recordedAudio.Length / (time * 0.001f);
            timeText.text = $"Time: {time} ms\nRate: {rate:F1}x";

            var text = res.Result;
            if (printLanguage)
                text += $"\n\nLanguage: {res.Language}";
            
            outputText.text = text;
            UiUtils.ScrollDown(scroll);

            // --- NUEVO: Comprobar el texto al finalizar la grabación ---
            CheckColorCommand(res.Result);
        }
        
        private void OnLanguageChanged(int ind)
        {
            var opt = languageDropdown.options[ind];
            whisper.language = opt.text;
        }
        
        private void OnTranslateChanged(bool translate)
        {
            whisper.translateToEnglish = translate;
        }

        private void OnProgressHandler(int progress)
        {
            if (!timeText)
                return;
            timeText.text = $"Progress: {progress}%";
        }
        
        private void OnNewSegment(WhisperSegment segment)
        {
            if (!streamSegments || !outputText)
                return;

            _buffer += segment.Text;
            outputText.text = _buffer + "...";
            UiUtils.ScrollDown(scroll);

            // --- NUEVO: Comprobar el texto en tiempo real mientras hablas ---
            CheckColorCommand(segment.Text);
        }

        // --- NUEVA FUNCIÓN: Lógica para detectar la palabra "Rojo" ---
        private void CheckColorCommand(string text)
        {
            if (cuboRenderer == null) return;

            // Convertimos a minúsculas para que detecte "Rojo", "rojo" o "ROJO"
            string cleanText = text.ToLower();

            if (cleanText.Contains("rojo") || cleanText.Contains("red"))
            {
                cuboRenderer.material.color = Color.red;
            }
        }
    }
}

```

## Configuración de la Escena

Para conectar el código con los elementos visuales, se realizó una reestructuración de los componentes en el Editor de Unity:

- Se eliminó el componente Microphone Demo del objeto original `Whisper`.

- Se añadió el script Microphone Demo directamente al objeto Cube que ya estaba creado en la escena y que queremos que cambie de color.

- En el Inspector del Cube, dentro del componente Microphone Demo, se asignó la variable Cubo Renderer arrastrando el propio componente MeshRenderer del cubo.

- Se restablecieron las referencias a Whisper Manager y Microphone Record dentro del script para asegurar la comunicación con el sistema de audio.

## Resultado Final

Al ejecutar la aplicación, el usuario puede presionar "Record" y decir frases como "Quiero que el cubo sea rojo". El sistema transcribe el audio a texto, detecta la palabra clave y actualiza el material del cubo instantáneamente.

A continuación se ve como el funcionamiento de la aplicación en un dispositivo Android y como el cubo logra cambiar de color, no se puede apreciar que se dice la palabra `rojo` al ser un GIF, pero se puede ver la palabra `rojo`cuando el sistema lo detecta en el botón que por defecto tiene el texto `Record` y que se encuentra abajo a la derecha de la pantalla. 

![Video]()
