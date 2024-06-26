enable_testing()

# find_path(CATCH_INCLUDE_DIR NAMES catch.hpp PATH_SUFFIXES catch2)
find_package(Catch2 CONFIG REQUIRED)# 

add_executable(test_catch
         test_catch.cpp)

target_compile_features(test_catch PUBLIC cxx_std_17)

target_include_directories(test_catch
        PUBLIC ../include
        PRIVATE ../src)

target_link_libraries(test_catch
        PRIVATE
        project_options
        project_warnings
        Catch2::Catch2
        Catch2::Catch2WithMain
        PUBLIC
)
add_test(AllCatchTests test_catch)


###########################################################
#Google testing
###########################################################

find_package(GTest CONFIG REQUIRED)

add_executable(test_gg
         test_gg.cpp)

target_compile_features(test_gg PUBLIC cxx_std_17)

target_include_directories(test_gg
        PUBLIC ../include
        PRIVATE ../src)

target_link_libraries(test_gg
                        PRIVATE 
                        project_options       
                        project_warnings 
                        GTest::gtest 
                        GTest::gtest_main) 
#                        GTest::gmock 
#                        GTest::gmock_main)
                        
add_test(AllGGTests test_gg)
#