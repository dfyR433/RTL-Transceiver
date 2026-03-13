##########################################################################################
## CMakeLists.txt for custom application ameba_test
##########################################################################################

set(private_sources)
set(private_includes)
set(private_definitions)
set(private_compile_options)

# Add your application sources
ameba_list_append(private_sources
    app_example.c
    monitor.c
    inject.c
)

# Private definitions (if any):
ameba_list_append(private_definitions
    USE_DWT_CYCCNT
)

# Private definitions (if any):
# ameba_list_append(private_definitions SOME_DEFINE)

# Private compile options (if any):
# ameba_list_append(private_compile_options -Wall)

# Add as an internal library with a unique name
ameba_add_internal_library(ameba_test
    p_SOURCES        ${private_sources}
    p_INCLUDES       ${private_includes}
    p_DEFINITIONS    ${private_definitions}
    p_COMPILE_OPTIONS ${private_compile_options}
)
##########################################################################################
