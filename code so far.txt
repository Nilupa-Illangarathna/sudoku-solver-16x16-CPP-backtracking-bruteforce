#include <bits/stdc++.h>
using namespace std;

typedef unsigned long mask_t;
int PUZZLE_LENGTH;
int PUZZLE_SIZE;
int BLOCK_SIZE;

#define IDX_EMPTY (-1)
#define IDX_NOT_SET (-2)

vector<mask_t> SOLVED_PUZZLE;
int totalRecursiveCalls = 0;

inline int at(int i, int j) {
    return i * PUZZLE_LENGTH + j;
}

inline pair<int, int> coord(int idx) {
    int i = idx / PUZZLE_LENGTH;
    int j = idx % PUZZLE_LENGTH;
    return make_pair(i, j);
}

inline mask_t mask(int value) {
    return value > 0 ? 1 << (value - 1) : 0;
}

inline int unmask(mask_t mask) {
    if (mask <= 0) return 0;
    int value = 1;
    while (mask >>= 1) value++;
    return value;
}

inline int countBits(mask_t mask) {
    int count = 0;
    while (mask) {
        mask &= (mask - 1);
        count++;
    }
    return count;
}

inline void printMatrix(vector<mask_t> const &matrix, const string &title, int indent = 0) {
    cout << title + ":\n";
    for (int i = 0; i < PUZZLE_LENGTH; i++) {
        cout << std::string((indent), ' ');
        for (int j = 0; j < PUZZLE_LENGTH; j++) {
            cout << setw(3) << unmask(matrix[at(i, j)]);
        }
        cout << "\n";
    }
}

inline void dumpMatrix(vector<mask_t> const &matrix, const string &filename, bool solved = true) {
    string buffer = "";
    if (solved) {
        for (int i = 0; i < PUZZLE_LENGTH; i++) {
            for (int j = 0; j < PUZZLE_LENGTH; j++) {
                buffer += (to_string(unmask(matrix[at(i, j)])) + " ");
            }
            buffer += "\n";
        }
    } else {
        buffer = "No Solution";
    }
    ofstream file;
    file.open(filename);
    file << buffer;
    file.close();
}

inline void updateAllowed(vector<mask_t> &puzzle, vector<mask_t> &allowed, int idx) {
    allowed[idx] = 0;

    pair<int, int> pos = coord(idx);
    int i = pos.first, j = pos.second;

    mask_t safeMask = ~(puzzle[idx]);

    int block_i = i - i % BLOCK_SIZE, block_j = j - j % BLOCK_SIZE;
    for (int _i = 0; _i < BLOCK_SIZE; _i++) {
        for (int _j = 0; _j < BLOCK_SIZE; _j++) {
            allowed[at(block_i + _i, block_j + _j)] &= safeMask;
        }
    }

    for (int _ = 0; _ < PUZZLE_LENGTH; _++) {
        allowed[at(_, j)] &= safeMask;
        allowed[at(i, _)] &= safeMask;
    }
}

inline int getToFillIndex(vector<mask_t> &puzzle, vector<mask_t> &allowed) {
    int minAllowedCount = BLOCK_SIZE;

    int allowedCount = -1;
    int toFill = IDX_EMPTY;
    for (int i = 0; i < PUZZLE_SIZE; i++) {
        if (puzzle[i] == 0) {
            allowedCount = countBits(allowed[i]);
            if (allowedCount == 1) {
                return i;
            }
            if (allowedCount < minAllowedCount) {
                toFill = i;
                minAllowedCount = allowedCount;
            }
        }
    }
    return toFill;
}

inline bool solvePuzzleRecursive(vector<mask_t> puzzle, vector<mask_t> allowed, int idx = IDX_NOT_SET, int value = 0) {
    totalRecursiveCalls++;
    if (idx == IDX_EMPTY) {
        SOLVED_PUZZLE = puzzle;
        return true;
    }
    if (idx == IDX_NOT_SET) {
        idx = getToFillIndex(puzzle, allowed);
    }
    if (value > 0) {
        puzzle[idx] = mask(value);
        updateAllowed(puzzle, allowed, idx);
        idx = getToFillIndex(puzzle, allowed);
    }

    for (int val = 1; val <= PUZZLE_LENGTH; val++) {
        if ((allowed[idx] & mask(val)) > 0) {
            if (solvePuzzleRecursive(puzzle, allowed, idx, val)) {
                return true;
            }
        }
    }
    return false;
}

inline bool solvePuzzle(vector<mask_t> &puzzle, vector<mask_t> &allowed) {
    for (int idx = 0; idx < PUZZLE_SIZE; idx++) {
        if (puzzle[idx] > 0) {
            allowed[idx] = 0;
            updateAllowed(puzzle, allowed, idx);
        }
    }
    return solvePuzzleRecursive(puzzle, allowed);
}

int main(int argc, char *argv[]) {
    auto programTimeStart = chrono::steady_clock::now();

    int input, nZeros = 0;
    vector<mask_t> puzzle;
    puzzle.reserve(257);
    string inputfile;

    if (argc > 1 && freopen(argv[1], "rt", stdin) != nullptr) {
        cout << "$ [START] " << argv[1] << "\n\n";
        inputfile = argv[1];

        int size = 0;
        while (cin >> input) {
            puzzle.push_back(mask(input));
            size++;
            if (input == 0) {
                nZeros++;
            }
        }
        if (size == 256) {
            BLOCK_SIZE = 4;
            PUZZLE_LENGTH = 16;
            PUZZLE_SIZE = 256;
        } else if (size == 81) {
            BLOCK_SIZE = 3;
            PUZZLE_LENGTH = 9;
            PUZZLE_SIZE = 81;
        } else {
            printf("$ No enough data, exiting!");
            exit(0);
        }
    } else {
        printf("$ Input file not loaded, exiting!");
        exit(0);
    }

    vector<mask_t> allowed(PUZZLE_SIZE, mask(PUZZLE_LENGTH + 1) - 1);

    auto solvingTimeStart = chrono::steady_clock::now();
    bool solved = solvePuzzle(puzzle, allowed);
    auto solvingTimeEnd = chrono::steady_clock::now();

    printMatrix(puzzle, "Input");
    string outputfile = inputfile.substr(inputfile.find_last_of("/\\") + 1);
    outputfile = outputfile.substr(0, outputfile.find_last_of(".")) + "_output.txt";

    if (solved) {
        printMatrix(SOLVED_PUZZLE, "\nOutput");
        dumpMatrix(SOLVED_PUZZLE, outputfile);
    } else {
        cout << "$ There is no solution for the given puzzle " << endl;
        dumpMatrix(SOLVED_PUZZLE, outputfile, false);
    }

    cout << setprecision(3) << "\n\n";
    cout << "$ Total Zeros in input : " << nZeros << "\n";
    cout << "$ Total Recursive calls: " << totalRecursiveCalls << "\n";
    cout << "$ Calculation Time     : " << chrono::duration<double>(solvingTimeEnd - solvingTimeStart).count() << " s\n";

    auto programTimeEnd = chrono::steady_clock::now();
    cout << "$ Total Program Time   : " << chrono::duration<double>(programTimeEnd - programTimeStart).count() << " s\n";
    cout << "$ [DONE]\n";

    return 0;
}
