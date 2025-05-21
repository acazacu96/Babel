cmake_minimum_required(VERSION 3.10)
project(babel_project)

# Create the executable
add_executable(babel_function function.cc)

# Set C++ standard (optional but recommended)
set_target_properties(babel_function PROPERTIES
        CXX_STANDARD 20
        CXX_STANDARD_REQUIRED ON
)