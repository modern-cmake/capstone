# Regression testing
This directory contains a tool for regression testing core of Capstone

## Build
- Download [Cmocka](git://git.cryptomilk.org/projects/cmocka.git)
- Build Cmocka
```
cd cmocka_dir
mkdir build
cd build
cmake ..
make
sudo make isntall
```
- Then build `cstest`
```
cd suite/cstest
make
```

## Usage
- Test for all closed issues
```
cd suite/cstest
./build/cstest -f ./issues.cs
```
- Test for some input from LLVM
```
cd suite/cstest
./build/cstest -f ../MC/AArch64/basic-a64-instructions.s.cs
```
- Test for all cs file in a folder
```
cd suite/cstest
./build/cstest -d ../MC
```
- Test all
```
cd suite/cstest
make cstest
```