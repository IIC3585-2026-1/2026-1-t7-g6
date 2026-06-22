# Astro Starter Kit: Basics

```sh
npm create astro@latest -- --template basics
```

> 🧑‍🚀 **Seasoned astronaut?** Delete this file. Have fun!

## 🚀 Project Structure

Inside of your Astro project, you'll see the following folders and files:

```text
/
├── public/
│   └── favicon.svg
├── src
│   ├── assets
│   │   └── astro.svg
│   ├── components
│   │   └── Welcome.astro
│   ├── layouts
│   │   └── Layout.astro
│   └── pages
│       └── index.astro
└── package.json
```

To learn more about the folder structure of an Astro project, refer to [our guide on project structure](https://docs.astro.build/en/basics/project-structure/).

## 🧞 Commands

All commands are run from the root of the project, from a terminal:

| Command                   | Action                                           |
| :------------------------ | :----------------------------------------------- |
| `npm install`             | Installs dependencies                            |
| `npm run dev`             | Starts local dev server at `localhost:4321`      |
| `npm run build`           | Build your production site to `./dist/`          |
| `npm run preview`         | Preview your build locally, before deploying     |
| `npm run astro ...`       | Run CLI commands like `astro add`, `astro check` |
| `npm run astro -- --help` | Get help using the Astro CLI                     |

## Decisión técnica: polling cada 20 segundos

Inicialmente, la isla Svelte consultaba el precio de la criptomoneda seleccionada cada 5 segundos. Durante las pruebas detectamos que esta frecuencia podía superar el límite de solicitudes de la API pública de CoinGecko y provocar errores de *rate limiting*.

Por este motivo, cambiamos el intervalo de polling a 20 segundos. Con esto reducimos la cantidad de solicitudes, mantenemos la actualización periódica de los precios y hacemos que el dashboard sea más estable. Si una consulta falla, el componente muestra un mensaje de error y vuelve a intentarlo automáticamente en el siguiente ciclo.

## Uso de inteligencia artificial

Utilizamos inteligencia artificial como herramienta de apoyo durante el desarrollo. Nos ayudó a separar las responsabilidades del proyecto según la arquitectura de islas: Astro para la estructura estática, Vue para los filtros y la configuración, y Svelte para los datos en tiempo real y su visualización.

También la utilizamos para aprender el funcionamiento básico de Vue y Svelte, comprender sus ciclos de vida y conectar componentes de frameworks distintos. En particular, nos orientó en la comunicación mediante el evento global `crypto-changed`, emitido por Vue y escuchado por Svelte, además del manejo del polling y la limpieza de intervalos y peticiones.

Las propuestas generadas fueron revisadas, probadas y adaptadas a las necesidades del proyecto antes de incorporarlas.

## Reflexión final

Consideramos que este trabajo fue una buena forma de entender la arquitectura de islas de manera práctica. Separar la página estática de los componentes interactivos permitió reconocer que no todo el sitio necesita cargar JavaScript al mismo tiempo y que cada framework puede utilizarse donde aporta mayores ventajas.

El desafío más interesante fue comunicar las islas Vue y Svelte sin depender de un estado global compartido. El uso de eventos personalizados resultó ser una solución sencilla y desacoplada. Además, los problemas de *rate limiting* mostraron que trabajar con APIs externas no consiste solamente en consumir datos también es necesario manejar errores, límites y limpieza de recursos. El resultado es un dashboard simple, pero suficientemente completo para demostrar generación estática, interactividad y actualización periódica de datos.

## 👀 Want to learn more?

Feel free to check [our documentation](https://docs.astro.build) or jump into our [Discord server](https://astro.build/chat).
