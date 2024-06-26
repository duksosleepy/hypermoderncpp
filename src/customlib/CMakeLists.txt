set(NAME "customlib")

set(HEADER_LIST ../../include/customlib/lib.hpp)

#${CMAKE_CURRENT_SOURCE_DIR} = C:/Users/Khoi Nguyen/Desktop/cpptest/src/customlib/
# Make an automatic library - will be static or dynamic based on user setting
add_library(${NAME} STATIC
        lib.cpp ${HEADER_LIST})  #SomeLibrary/SomeFile.hpp)
        
# We need this directory, and users of our library will need it too
target_include_directories(
  ${NAME} PUBLIC ../../include) #PUBLIC include PRIVATE src

# All users of this library will need at least C++17
target_compile_features(${NAME} PRIVATE cxx_std_17)

# This depends on (header only) 
target_link_libraries(${NAME} 
                PRIVATE
                project_options 
                project_warnings
                ) #PUBLIC clara::calra)

source_group(
                TREE "${PROJECT_SOURCE_DIR}/include"
                PREFIX "Header Files"
                FILES ${HEADER_LIST})
# install(TARGETS ${NAME}
#                 PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}   
#                 RUNTIME DESTINATION       ${CMAKE_INSTALL_BINDIR}  # For executables
#                 LIBRARY DESTINATION       ${CMAKE_INSTALL_LIBDIR}  # For shared libraries
#                 INCLUDES DESTINATION      ${CMAKE_INSTALL_INCLUDEDIR}
#                 ARCHIVE DESTINATION       ${CMAKE_INSTALL_SHAREDSTATEDIR}  # For static libraries
#         )