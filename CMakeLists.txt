cmake_minimum_required(VERSION 3.15)
project(prosail_model_fortran LANGUAGES Fortran C) # Langages utilisés

# Trouvez Python
find_package(Python COMPONENTS Interpreter Development REQUIRED)

# Obtenir le répertoire des en-têtes NumPy en exécutant un script Python
execute_process(
  COMMAND ${Python_EXECUTABLE} -c "import numpy; print(numpy.get_include())"
  OUTPUT_VARIABLE NumPy_INCLUDE_DIR
  OUTPUT_STRIP_TRAILING_WHITESPACE
  RESULT_VARIABLE _numpy_include_result
  ERROR_QUIET # Pour ne pas afficher d'erreur si numpy n'est pas là, on le gère après
)

if(NOT _numpy_include_result EQUAL "0" OR NOT NumPy_INCLUDE_DIR)
  message(FATAL_ERROR "Impossible d'obtenir le répertoire d'inclusion de NumPy via Python. Assurez-vous que NumPy est installé dans l'environnement de build. Output: ${NumPy_INCLUDE_DIR}")
endif()

message(STATUS "Répertoire d'inclusion NumPy trouvé : ${NumPy_INCLUDE_DIR}") # Pour le débogage

# Ajoutez le répertoire d'inclusion de NumPy aux chemins d'inclusion de CMake
# Ceci est utile si vous avez du code C/C++ qui inclut directement des en-têtes NumPy
# ou si f2py a besoin d'un indice, bien que f2py le trouve souvent par lui-même.
include_directories(${NumPy_INCLUDE_DIR})

# Obtenir le suffixe d'extension Python (ex: .cpython-312-darwin.so)
execute_process(
  COMMAND ${Python_EXECUTABLE} -c "import sysconfig; print(sysconfig.get_config_var('EXT_SUFFIX'))"
  OUTPUT_VARIABLE PYTHON_EXTENSION_SUFFIX
  OUTPUT_STRIP_TRAILING_WHITESPACE
  RESULT_VARIABLE _python_ext_suffix_result
  ERROR_QUIET
)
if(NOT _python_ext_suffix_result EQUAL "0" OR NOT PYTHON_EXTENSION_SUFFIX)
  message(FATAL_ERROR "Impossible d'obtenir le suffixe d'extension Python via sysconfig. Output: ${PYTHON_EXTENSION_SUFFIX}")
endif()
message(STATUS "Suffixe d'extension Python trouvé : ${PYTHON_EXTENSION_SUFFIX}")

# Définition des fichiers source Fortran
set(FORTRAN_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/pyprosail/MODULE_PRO4SAIL.f90
    ${CMAKE_CURRENT_SOURCE_DIR}/pyprosail/dataSpec_P5B.f90
    ${CMAKE_CURRENT_SOURCE_DIR}/pyprosail/LIDF.f90
    ${CMAKE_CURRENT_SOURCE_DIR}/pyprosail/dladgen.f
    ${CMAKE_CURRENT_SOURCE_DIR}/pyprosail/PRO4SAIL.f90
    ${CMAKE_CURRENT_SOURCE_DIR}/pyprosail/prospect_5B.f90
    ${CMAKE_CURRENT_SOURCE_DIR}/pyprosail/tav_abs.f90
    ${CMAKE_CURRENT_SOURCE_DIR}/pyprosail/volscatt.f90
    ${CMAKE_CURRENT_SOURCE_DIR}/pyprosail/PyPROSAIL.f90
)

# Nom du module (doit correspondre à -m _prosail_model)
set(MODULE_NAME _prosail_model)

# Définir le chemin attendu pour le module compilé
set(F2PY_WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
file(MAKE_DIRECTORY ${F2PY_WORKING_DIRECTORY})
set(COMPILED_MODULE_PATH ${F2PY_WORKING_DIRECTORY}/${MODULE_NAME}${PYTHON_EXTENSION_SUFFIX})

# Commande f2py
add_custom_command(
    OUTPUT ${COMPILED_MODULE_PATH}
    COMMAND ${Python_EXECUTABLE} -m numpy.f2py -c
            -m ${MODULE_NAME}
            ${FORTRAN_SOURCES}
            --build-dir ${CMAKE_CURRENT_BINARY_DIR}/f2py_build_temp
    DEPENDS ${FORTRAN_SOURCES}
    WORKING_DIRECTORY ${F2PY_WORKING_DIRECTORY}
    COMMENT "Construction du module Fortran ${MODULE_NAME} avec f2py"
    VERBATIM
)

# La cible personnalisée dépend du chemin de sortie mis à jour
add_custom_target(f2py_build ALL DEPENDS ${COMPILED_MODULE_PATH})

# Installer le module depuis CMAKE_CURRENT_SOURCE_DIR vers le paquet pyprosail
install(FILES ${COMPILED_MODULE_PATH}
        DESTINATION pyprosail
        COMPONENT Runtime)

message(STATUS "Chemin attendu du module compilé : ${COMPILED_MODULE_PATH}")