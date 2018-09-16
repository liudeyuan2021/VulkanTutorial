## Structure générale

Dans le chapitre précédent nous avons créé un projet Vulkan avec une configuration solide et nous l'avons testé. Nous
recommençons ici à partir du code suivant :

```c++
#include <vulkan/vulkan.h>

#include <iostream>
#include <stdexcept>
#include <functional>
#include <cstdlib>

class HelloTriangleApplication {
public:
    void run() {
        initVulkan();
        mainLoop();
        cleanup();
    }

private:
    void initVulkan() {

    }

    void mainLoop() {

    }

    void cleanup() {

    }
};

int main() {
    HelloTriangleApplication app;

    try {
        app.run();
    } catch (const std::exception& e) {
        std::cerr << e.what() << std::endl;`
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

Nous incluons ici d'abord le header Vulkan du SDK, qui fournit les fonctions, les structures et les énumérations.
`stdexcept` et `iostream` nous permettront de reporter et de traiter les erreurs. Le header `functional` nous servira
pour l'écriture d'une lambda dans la section sur la gestion des ressources. `cstdlib` nous fournit `EXIT_FAILURE` et
`EXIT_SUCCESS` (optionels).

Le programme est écrit à l'intérieur d'une classe, dans laquelle nous stockerons les objets Vulkan. Nous avons également
 une fonction pour la création de chacun de ces objets. Une fois toute l'initialisation réalisée, nous entrons dans la
boucle principale, qui attendra que nous fermions la fenêtre pour quitter le programme, après avoir libéré toutes les
ressources que nous avons allouées grâce à la fonction cleanup.

Si nous rencontrons une quelconque erreur lors de l'exécution nous soulèverons une `std::runtime_error` comportant un
message descriptif, qui sera affiché sur le terminal depuis la fonction `main`. Afin de s'assurer que nous récupérons
bien toutes les erreurs, nous utilisons `std::exception` dans le `catch`. Nous verrons bientôt que la requête de
certaines extensions peut mener à soulever des exceptions.

À peu près tous les chapitres à partir de celui-ci introduirons une nouvelle fonction appelée dans `initVulkan` et un
nouvel objet Vulkan qui sera justement créé par cette fonction. Nous le détruirons également dans `cleanup`.

## Gestion des ressources

Comme pour une quelconque ressource explicitement allouée par `new` doit être explicitement libérée par `delete`, nous
devrons explicitement détruire toutes les ressources Vulkan que nous allouerons. Il est possible d'exploiter des
fonctioinnalités du C++ pour s'aquitter automatiquement de cela. Ces possibilités sont localisées dans `<memory>` si
vous désirez les utiliser. Cependant nous resterons explicites pour toutes les opérations dans ce tutoriel, car la
puissance de Vulkan réside spécifiquement dans la clareté de ce que le programmeur veut. De plus cela nous permettra de
bien comprendre les durées de vie des objets.

Après avoir suivi ce tutoriel vous pourrez parfaitement implémenter une gestion automatique des ressources en
spécialisant `std::shared_ptr` par exemple. L'utilisations du [RAII](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)
à votre avantage est toujours recommandé avec le C++ pour de gros programmes Vulkan, mais il est quand même bon de
connaître les détails de l'implémentation.

Les objets Vulkan peuvent être crées de deux manières. Soit ils sont directement crées avec une fonction du type
`vkCreateXXX`, soit il sont récupérés à partir d'un autre objet avec une fonction du type `vkAllocateXXX`. Après vous
être assuré qu'il n'est plus utilisé où que ce soit, vous devrezle détruire en utilisant les fonctions du type
`vkDestroyXXX` ou `vkFreeXXX`, respectivement. Les paramètres de ces fonctions varient sauf pour l'un d'entre eux :
`pAllocator`. Ce paramètre optionnel vous permet de spécifier un callback sur un allocateur de mémoire. Nous
n'utiliserons jamais ce paramètre et l'indiquerons donc toujours nullptr.

## Intégrer GLFW

Vulkan marche très bien sans fenêtre si vous voulez l'utiliser pour du rendu sans écran, mais c'est tout de même plus
exitant d'afficher quelque chose! Remplacez d'abord la ligne `#include <vulkan/vulkan.h>` par :

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
```

GLFW va ainsi automatiquement inclureses propres définitions des fonctions Vulkan et vous fournir le header Vulkan.
Ajoutez une fonction `initWindow` et appelez-la depuis `run` avant les autres appels. Nous utiliserons cette fonction
pour initialiser GLFW et créer une fenêtre.

```c++
void run() {
    initWindow();
    initVulkan();
    mainLoop();
    cleanup();
}

private:
    void initWindow() {

    }
```

Le premier appel dans `initWindow` doit être `glfwInit()`, ce qui initialize la librairie. Dans la mesure où GLFW a été
créer pour fonctionner avec OpenGL, nousdevons lui demander de ne pas créer de contexte OpenGL avec l'appel suivant :

```c++
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
```

Dans la mesure où redimensionner une fenêtre n'est pas chose aisée avec Vulkan, nous verrons cela plus tard et
l'interdisons pour l'instant

```c++
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
```

Il ne nous reste plus qu'à créer la fenêtre. Ajoutez un membre privé `GLFWWindow *m_window` pour en stocker une
référence, et initialisez la ainsi :

```c++
window = glfwCreateWindow(800, 600, "Vulkan", nullptr, nullptr);
```

Les trois premiers paramètres indique respectivement la largeur, la hauteur et le titre de la fenêtre. Le quatrième vous
 permet optionnellement de spécifier un moniteur sur lequel ouvrir la fenêtre, et le cinquième est spécifique à OpenGL.

Nous devrions plutôt utiliser des constantes pour la hauteur et la largeur dans la mesure où nous aurons besoin de ces
valeurs dans le futur. J'ai donc ajouté ceci au-dessus de la définition de la classe `HelloTriangleApplication` :

```c++
const int WIDTH = 800;
const int HEIGHT = 600;
```

et remplacé la création de la fenêtre par :

```c++
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
```

Vous avez maintenant une fonction `initWindow` semblable à ceci :

```c++
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
}
```

Pour s'assurer que l'application tourne jusqu'à ce que soit une erreur soit un click sur la croix ne l'interrompe, nous
devons écrire une petite boucle de gestion d'évènements :

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }
}
```

Ce code est relativement simple. GLFW récupère tous les évènements disponible, puis vérifie qu'aucun d'entre eux ne
correspond à une demande de fermeture de fenêtre. Nous appellerons aussi ici une fonction pour afficher notre triangle.

Une fois la requête pour la fermeture de la fenêtre récupérée, nous devons détruire toutes les ressources allouées et
quitter GLFW. Voici notre première version de la fonction `cleanup` :

```c++
void cleanup() {
    glfwDestroyWindow(window);

    glfwTerminate();
}
```

Si vous lancez l'application, vous devriez voir une fenêtre appelée "Vulkan" se fermant en cliquant sur la croix.
Maintenant que nous avons une base pour notre application Vulkan, [créons notre premier objet Vulkan!](!Drawing_a_triangle/Setup/Instance)!

[Code C++](/code/00_base_code.cpp)