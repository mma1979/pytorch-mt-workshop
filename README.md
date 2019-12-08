
{
 "cells": [
  {
   "cell_type": "code",
   "execution_count": 23,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Requirements\n",
    "from __future__ import unicode_literals, print_function, division\n",
    "from io import open\n",
    "import unicodedata\n",
    "import string\n",
    "import re\n",
    "import random\n",
    "\n",
    "import torch\n",
    "import torch.nn as nn\n",
    "from torch import optim\n",
    "import torch.nn.functional as F\n",
    "\n",
    "device = torch.device(\"cuda\" if torch.cuda.is_available() else \"cpu\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "We’ll need a unique index per word to use as the inputs and targets of the networks later. To keep track of all this we will use a helper class called Lang which has word → index (word2index) and index → word (index2word) dictionaries, as well as a count of each word word2count to use to later replace rare words."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 24,
   "metadata": {},
   "outputs": [],
   "source": [
    "SOS_token = 0\n",
    "EOS_token = 1\n",
    "\n",
    "\n",
    "class Lang:\n",
    "    def __init__(self, name):\n",
    "        self.name = name\n",
    "        self.word2index = {}\n",
    "        self.word2count = {}\n",
    "        self.index2word = {0: \"SOS\", 1: \"EOS\"}\n",
    "        self.n_words = 2  # Count SOS and EOS\n",
    "\n",
    "    def addSentence(self, sentence):\n",
    "        for word in sentence.split(' '):\n",
    "            self.addWord(word)\n",
    "\n",
    "    def addWord(self, word):\n",
    "        if word not in self.word2index:\n",
    "            self.word2index[word] = self.n_words\n",
    "            self.word2count[word] = 1\n",
    "            self.index2word[self.n_words] = word\n",
    "            self.n_words += 1\n",
    "        else:\n",
    "            self.word2count[word] += 1"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "The files are all in Unicode, to simplify we will turn Unicode characters to ASCII, make everything lowercase, and trim most punctuation."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 25,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Turn a Unicode string to plain ASCII, thanks to\n",
    "# https://stackoverflow.com/a/518232/2809427\n",
    "def unicodeToAscii(s):\n",
    "    return ''.join(\n",
    "        c for c in unicodedata.normalize('NFD', s)\n",
    "        if unicodedata.category(c) != 'Mn'\n",
    "    )\n",
    "\n",
    "# Lowercase, trim, and remove non-letter characters\n",
    "\n",
    "\n",
    "def normalizeString(s):\n",
    "    s = unicodeToAscii(s.lower().strip())\n",
    "    s = re.sub(r\"([.!?])\", r\" \\1\", s)\n",
    "    s = re.sub(r\"[^a-zA-Z.!?]+\", r\" \", s)\n",
    "    return s"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "To read the data file we will split the file into lines, and then split lines into pairs. The files are all English → Other Language, so if we want to translate from Other Language → English I added the reverse flag to reverse the pairs."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 26,
   "metadata": {},
   "outputs": [],
   "source": [
    "def readLangs(lang1, lang2, reverse=False):\n",
    "    print(\"Reading lines...\")\n",
    "\n",
    "    # Read the file and split into lines\n",
    "    lines = open('data/%s-%s.txt' % (lang1, lang2), encoding='utf-8').\\\n",
    "        read().strip().split('\\n')\n",
    "\n",
    "    # Split every line into pairs and normalize\n",
    "    pairs = [[normalizeString(s) for s in l.split('\\t')] for l in lines]\n",
    "\n",
    "    # Reverse pairs, make Lang instances\n",
    "    if reverse:\n",
    "        pairs = [list(reversed(p)) for p in pairs]\n",
    "        input_lang = Lang(lang2)\n",
    "        output_lang = Lang(lang1)\n",
    "    else:\n",
    "        input_lang = Lang(lang1)\n",
    "        output_lang = Lang(lang2)\n",
    "\n",
    "    return input_lang, output_lang, pairs"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 27,
   "metadata": {},
   "outputs": [],
   "source": [
    "MAX_LENGTH = 10\n",
    "\n",
    "eng_prefixes = (\n",
    "    \"i am \", \"i m \",\n",
    "    \"he is\", \"he s \",\n",
    "    \"she is\", \"she s \",\n",
    "    \"you are\", \"you re \",\n",
    "    \"we are\", \"we re \",\n",
    "    \"they are\", \"they re \"\n",
    ")\n",
    "\n",
    "\n",
    "def filterPair(p):\n",
    "    return len(p[0].split(' ')) < MAX_LENGTH and \\\n",
    "        len(p[1].split(' ')) < MAX_LENGTH and \\\n",
    "        p[1].startswith(eng_prefixes)\n",
    "\n",
    "\n",
    "def filterPairs(pairs):\n",
    "    return [pair for pair in pairs if filterPair(pair)]"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "The full process for preparing the data is:\n",
    "\n",
    "* Read text file and split into lines, split lines into pairs\n",
    "* Normalize text, filter by length and content\n",
    "* Make word lists from sentences in pairs"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 28,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Reading lines...\n",
      "Read 135842 sentence pairs\n",
      "Trimmed to 10599 sentence pairs\n",
      "Counting words...\n",
      "Counted words:\n",
      "fra 4345\n",
      "eng 2803\n",
      "['elle est completement sourde de l oreille gauche .', 'she is completely deaf in her left ear .']\n"
     ]
    }
   ],
   "source": [
    "def prepareData(lang1, lang2, reverse=False):\n",
    "    input_lang, output_lang, pairs = readLangs(lang1, lang2, reverse)\n",
    "    print(\"Read %s sentence pairs\" % len(pairs))\n",
    "    pairs = filterPairs(pairs)\n",
    "    print(\"Trimmed to %s sentence pairs\" % len(pairs))\n",
    "    print(\"Counting words...\")\n",
    "    for pair in pairs:\n",
    "        input_lang.addSentence(pair[0])\n",
    "        output_lang.addSentence(pair[1])\n",
    "    print(\"Counted words:\")\n",
    "    print(input_lang.name, input_lang.n_words)\n",
    "    print(output_lang.name, output_lang.n_words)\n",
    "    return input_lang, output_lang, pairs\n",
    "\n",
    "\n",
    "input_lang, output_lang, pairs = prepareData('eng', 'fra', True)\n",
    "print(random.choice(pairs))"
   ]
  },
  {
   "attachments": {
    "encoder-network.png": {
     "image/png": "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"
    }
   },
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# The Encoder\n",
    "The encoder of a seq2seq network is a RNN that outputs some value for every word from the input sentence. For every input word the encoder outputs a vector and a hidden state, and uses the hidden state for the next input word.\n",
    "![encoder-network.png](attachment:encoder-network.png)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 29,
   "metadata": {},
   "outputs": [],
   "source": [
    "class EncoderRNN(nn.Module):\n",
    "    def __init__(self, input_size, hidden_size):\n",
    "        super(EncoderRNN, self).__init__()\n",
    "        self.hidden_size = hidden_size\n",
    "\n",
    "        self.embedding = nn.Embedding(input_size, hidden_size)\n",
    "        self.gru = nn.GRU(hidden_size, hidden_size)\n",
    "\n",
    "    def forward(self, input, hidden):\n",
    "        embedded = self.embedding(input).view(1, 1, -1)\n",
    "        output = embedded\n",
    "        output, hidden = self.gru(output, hidden)\n",
    "        return output, hidden\n",
    "\n",
    "    def initHidden(self):\n",
    "        return torch.zeros(1, 1, self.hidden_size, device=device)"
   ]
  },
  {
   "attachments": {
    "decoder-network.png": {
     "image/png": "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"
    }
   },
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# The Decoder\n",
    "The decoder is another RNN that takes the encoder output vector(s) and outputs a sequence of words to create the translation.\n",
    "![decoder-network.png](attachment:decoder-network.png)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 30,
   "metadata": {},
   "outputs": [],
   "source": [
    "class DecoderRNN(nn.Module):\n",
    "    def __init__(self, hidden_size, output_size):\n",
    "        super(DecoderRNN, self).__init__()\n",
    "        self.hidden_size = hidden_size\n",
    "\n",
    "        self.embedding = nn.Embedding(output_size, hidden_size)\n",
    "        self.gru = nn.GRU(hidden_size, hidden_size)\n",
    "        self.out = nn.Linear(hidden_size, output_size)\n",
    "        self.softmax = nn.LogSoftmax(dim=1)\n",
    "\n",
    "    def forward(self, input, hidden):\n",
    "        output = self.embedding(input).view(1, 1, -1)\n",
    "        output = F.relu(output)\n",
    "        output, hidden = self.gru(output, hidden)\n",
    "        output = self.softmax(self.out(output[0]))\n",
    "        return output, hidden\n",
    "\n",
    "    def initHidden(self):\n",
    "        return torch.zeros(1, 1, self.hidden_size, device=device)"
   ]
  },
  {
   "attachments": {
    "1152PYf.png": {
     "image/png": "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"
    }
   },
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Attention Decoder\n",
    "If only the context vector is passed betweeen the encoder and decoder, that single vector carries the burden of encoding the entire sentence.\n",
    "![1152PYf.png](attachment:1152PYf.png)"
   ]
  },
  {
   "attachments": {
    "attention-decoder-network.png": {
     "image/png": "iVBORw0KGgoAAAANSUhEUgAAAYsAAAJ1CAYAAAAokNx/AAAAAXNSR0IArs4c6QAAQABJREFUeAHsXQdgVdX5/4WQhIQkQCAkhI3sIQoyXAiiuPcerdtqh7WO/q1ttVqtWm3rbGtddY+6t+BA0VaRKaAgMsIKCRBWIDv8v989uXkvj5e8l+SNe+/7Pnh5995z7hm/c9/5ne/7zjk3ac+ePVVQUQQUAUVAEVAEmkEgSchiTzPhGqQIKAKKgCKgCKCdYqAIKAKKgCKgCIRCQMkiFEIarggoAoqAIqCahT4DioAioAgoAqERCK5Z7KkLfafGiD0CdbWxz1NzjC0C+tuLLd6aW9gItA8aM0k4ZNaNwIb/BQ3Wi3FAYMwvgAHHS8bJccg8wlm+fBRQq5Pw9kK150HAIbfvdTlmF7Z8B3z405hlpxk5EIHkVOD0D4IWLDhZMOrmRcC6mUFv0otxQGDQKXHINEpZrvtMyKIiSom7ONnUrPgWvmqH/ubj2wLxzz25Q5NlCG6GajK6BigCioAioAgkIgJKFonY6lpnRUARUARaiICSRQsB0+iKgCKgCCQiAkoWidjqWmdFQBFQBFqIgCfJ4psNwN2fNI3EoiLgz82EF+8EbpkOlFUGTyNUePC79Gq0EHhhPvD2t9FKPXi6O9Q/HxyYOF598HNg9po4FiDMrCP97EQ6vaaq4UmyWChkcVczZECyuGdmU5AAG4Us/tAMWYQKbzplDYkGAi8tBN6VWZ+xkp+/Ctw/K1a5aT7hIvCAkMWXheHGjk+8SD87fO6PfiQ2dfEkWYSC7twxQMktoWJpuFsQePVC4O+nxa60X7lg9Bo7NDSnliAQ6Wdn8UZgV4yWLDW9ziJMBKplUfFPXwGuOhR4WNbwFW4Fpg4CLp0AZKYBlTUA2fTyicCdMtrfp6t8Hwu0E5p6ao4xH1RInCn7AL84BGgva86oTqZJyS6Te2z5VkC5T67fdxLQIcW+2vz33HVmBFi625Tpl1LGpCTgKxl9PC+mi3tPNvdXSf7/+hKY8T1Q0Ak4fljjdEOFM/aC9cBDX5j6D88Dfj3FpMUw1mdQLrB+O/DmEim/1I34HDGYoSptRYC4d5Lp4eePNSPLj5YDRw0B/vFfoEi0xKkDgasnyXJGeeY48mQ7H9zPtHmqtMVpo4CTRppSlFfLs/wacMPh8qx2M9fWbTNmybtPAB77ClhdCrwh7cj0fjO1raV37/00f9CcO2ctkJsJXDjO/M5YI+LcXDvYtX5a+oDp0h78XZ40wnzYB1D+u9r0KRtk+Qd/U9dNBnp3ZoiR6cuAFxcAO8RczN9ToDRXvi9Wybpj+QzIAZ6ZJ8+ODCDP3C8whb3PaYKmVYLWi7ws88zxWaO09NlhfZrrO5nmvZ9JP5LduGy/fx+YvI/0lfL8vfKNLI2R5/Oyl6Qtjge6ZABPfg28thiokr55dA/g+ilAjlxvq0h2bZNa2Rnk0dnAkQ8bsE6UBn9QfrwXvABw83OSCcPPfVYIQB6CbeWGKH75OnDtm9KJyg/yoH5S0ZmycPApUxaSwW/eNffapWNnvnxT+ESxXR7kc54BRglYw+RBu1HSs01TP2wBnp5rpwz8TMjstg9NOUgM5z3nC+NRqPCPpXM68AHxcVQBZ4wWMloD7PsXWQAv5ED5YBnwk5eBf0sjkkhrBZejHgGWbDTh+rdtCLDT+Fx++BQ+I/yB/UjakB3LxD7S9u8Bt0v72uH3zAR+/DwwXsLy5QfPZ5ODBwrbn89rcZk5598tu8213dK+Q7sDHWUQ1FMGFezAElV2VQJj/wa8t9QQLTuu4x6V35V0/pRQ7cA4N38gxCz9QA/pDA/oJYPF12TAJQRPeUvI+NCHAP6OSeZfrJbf8j3Ayi1WMN6XfE98XPoYOeW9F79oBmomVEbbIcr3vTwnf5Pn5LfybHRON32XfW9T31vlORgjdX6XeY8A6iTzEx4zgxLe09JnJ1TfyTRpZvqfEK+/vLQAoEbRPRPoJc94Rqp5llOlf+UA/FfSrx7aHzhrNPDxDzIo/af/3a0/lnFVZOTooTLqOsukNVYab+y9wKcrpCF7m2un7wvccZw5ZkORUJ45Vzr0/X3hg+4097CSJBM+ECdIo9QI4TwvAN1zvIkbzt8aIbEXzpfGlbJQqPHMFOA4YvQXOsMfk85h8XXy4883ISPk+9q3zHGocMa6TuIeI/V/XvKjUCMa81fgTx9JPU811zpKg8680hDlzw6Shv6DGXkxL5XIIrBZftQfC9YcKFDW7zDaxE3TzDlHomyrY4eZ8+Qko03Yz6K5GvzvccONP4sdlK2NBI/p7av3f260tjlXi1Ynne2V8kwPzpVR7NtmtM3aN9cORdImt86Q37u00yQZJVM4guaImINMksi50jc8LX0E5QpJv//twO+kc39O2u7qN2QQMBWw2/RM6TMG32Xi8m845SuRAcF7l/r6CN/dwY/u+BjYKc/OqhsBaqQ/P8QMGm54x2hVwe/yXQ18diqqTVhTfefkgb57gx1xEDxBBjw/bPZZYUiqfDavOcxoa4cOEC1YMKWFh9aatkgbb/dlzc7Slv17ilraEZi33kcWE/raoUZt5QPx9VqjztkhmdKhUqU9TB4eksuz8wxZcGROYE+Ta+EKTT37Ffhij5EyUd0MlAVCFlQnbaJg+NGiVtpkESqcjbCwyIyO+NDYQhPFnHX2mWlAmt4o/ObIlJqISuQRSJe2t4mCqffp3HiWDDVcqvG2TJP2pol07TYgW7QGldAIcNYRtTJ2oLbQzEqNbJ18U5prh/nSN7Ad2JnZcrp0+PxwBL96qwy2jrVDzPfxQtTsC6g1LJcO8vBBvvD+XQ1Z2VfCKR/z9+8j7Hub+p4nv+cjJE8ShS0nSJnungksKxEyy7Gvtuy7qb4zFFkEy+Xs/WTg+ggw8A4zGCJmtnk/WPyWXKvvvlpyS/C4fbv4rtP+SNXOf+pp1wxfOE1RVFvZWO0krv1hpeyR9kXjjH1/Z4UxGXHkQHUrXOFI3u6ceQ/zCCYsC9VJ//cF2jZTxg8VTrso76d/xq4Hv48cbNRnO0+Wx184mlWJDgKBzwnbw799aV/3j2Pbc/2fV//4NKWqNEZgq/xuMlIaP/M0+/3mcN9vzR9j3u3fDrT9p8v97CsChb85Ss9s823/zZN2o+mGo3v+5mhx8JcUv94snPKxj/LvI/zTCnZMkxgHef7CgSaFpmVbWvrshOo7/dNjHvRFNCVTBor/9FqApEHTLGdKTbjf9GNN3RPudT+ODPeW4PFoG5vY14StkVEBmd82AQXeMbCb+COk0WliOqifCeVD8OQc3+iA2gVtmTQ/0Sn84RWBqUTmnGWkOkoNghoRhY45W0KFs+PhaJQqtP9IiHb0FCFDFechwJEvbeqcdED58HvxQwiZD5Hzyvofor/WR4e2SmME+BvmM37b0b4Od4X85r9YbawKjWPvfcaJLtuk8y0R0uhe3+FS26Af49lzZfQuv533JX3bRMUUqFXsJ7/RfPmtUauhY9wefW+S3/A3Rb582lo+X0q+I6ZJ07i/8JwD35H5hsAY1tJnp7m+k6Yj//RIkLbmxrwCuZYYZUl/dPux5kNMx99nsDtLCKQt4sfFbUnGOFao+nHEQG/9QHkYJg0InibZjz/MmyQenbw0MXFdw/+9LR1vB3MPRxycXUHHNEcsB/ULnlZbr9K+NyJP7KeSP2cV0EdBZ7otocIZj/ZaOsxJaiS9z1aIPfsJsdnuslPRb6chQHs5OyrOjKPP6sIDTKfH0W4vGT1y1hO1WjpUOfnBX6glf1ciNnuxuyeq/GSi6bRuERxpNiIWnBjCxZH+Zpqm8OHvmRMErnjFYMzOnv0GzddZ0gdw9uQz8puig5ezjB6R3+SX0r+csa9Jke313Hzjh2T+JBm/wT3aWr5g5b7iQPEPyPPABb98Nvg7f1jKRWc3O/XWPjt0SjfVd9IPREzpl6AVg+Zx9jG2tkGtmNjTD0wi4SwtTu7gYIhxNkoY/bck57ZKxDSLcb2F5f9u2HVkD+CdS03H76/a24XliPuNi4CLXjQzHKiu7iv30JnVTR4WWy6QB4Ikct1k+0rkvznz6u1LZObA08bOxxyuPcxoGjwOFc44N08zJrfTnzSjDKqm1082M6MYruIsBDjNlr6mPrcZ0wjt5Pee5CvjP06T2XzPyzTE35tn+P6T5Qco57acPFImYLwhncVK8XNInESUcX1EAzjPOJrv+th0lpwK/sAp4aFBU+8rF5iObfCdxoxL2/2tR5v77zgW4OwzznjiyJ0a/APSDmfvb8JvP8YMxo57zHSe9DvRL2lLW8tnp+P/TWvHY2eaCS0kNtaB0315zZaWPDvLbzB3NdV3MpR9Ec1JnPxDPyzN8/Sb2OY7DsjpHx1yl8ya+oVMDDgEoDbB2ZlckkDTH5cb2BON7HK25jtpj0jQG187QSj/7aBB/hepFaT/RmafXGHmrtOux4YNV7aXG+br6kcS4d4b6Xgc3VCFI0EEk1DhtG1Ts+J0tojLFNElRwvIfDmJ2+XedPmFy4MSB+HUTk4t3HyrTIkVzY8DFY4IA6Wuzsyiot08mF2b0yQ5Ygu0ywem06LzAfKbO0UKFy8p+kqG6xNbnDsd29QIwtEogiXOduDInH6/QCGp83fX1G+K/Q/NNP6DzMA02lq+wPTYY9IURFNYMFNzuM8OO/Jw+072K+ybmnre6OehD8YWuwzUkm1iscOa/eb7LK6WxIKINFHkhA9LS4iCOXPaXUvEXwULdh8bINiPO1jcwGuhyh4qnA9OUw91YF56Hn8Emhug8BnyXwAWWFo+6x6g7cBqteo80Onb0kSaaweSSHO/KQ7smhrc2eUIp3zsXOk0b07siS/sfJt7NsJ9dkh0toTqO21Huh0/8NufKBgWqgyB94dz3mayIHDWqEI6ylgIFwJxRWdTcsoosSOe3lSoXk90BNix8HlVUQT8EaC5+5//87/S+JimnqKbG19r61ms+862lrfNZEHmj+U+S5wWpqIItBYBrrDnR0UR8EeAvhLbX+J/PZrHse4721oX4UsVRUARUAQUAUWgeQSULJrHR0MVAUVAEVAEBAElC30MFAFFQBFQBEIioGQREiKNoAgoAoqAItC0gzuJPKJc4pxHRKadeUX02Qreki2aEB88ibZf1d982zF0cQrWbzN4+YMvyuOqE0c8uMELHexqRUUFOnSQBSVhSnl5OdLTW7jII8y0oxatjksym+b3qOXr4YTrZIJ9u9YuzIkGLi787UUDBk3TeQgEH0a4jCi4CH3evHlYv359WAhv2rQJc+bMQXW136qYsO6McyQliog2wJYtWzBr1izZQyfEaqyI5hoiMZf99kLUxlXB27dvt/oRRz0PDkIwOFk4qIDhFIU/+pqaGuTmyq5bYUi3bt3Qvn17rFu3LozYGkURUAQSAQH2ITt3yr4aKkER8ARZbNy4ETk5OUhNDW8DhiQZvfXs2RNFRUWgGUJFEVAEFAFFoHkEXE8WVVVVKC0tRX5+fvM1DQgtKCiwiIKEoaIIKAKKgCLQPAKuJwtqFTQpde3atfmaBoTynh49eqgpKgAXPVUEFAFFIBgCrieL4uJi5OXlyeStlk8t7dWrFyorK1FSIm+yUVEEFAFFQBFoEgFXkwVnL+zevbvFJigbjbS0NMspvnbtWvuSfisCioAioAgEQcDVZEETVFZWFjp2bP2e071790ZZWRm2bZN3qqokBAL0cS1dutSaak1zJI+XLVsGrtVRUQQUgeAIuJYsOIuJ6yVa6tgOhCEzMxNdunSBaheByHj3nNMjab7kAKG2ttYyQ3LgwckSKoqAIhAcAdeSxebNm63FVN27dw9esxZcpXbB0eauXfJ+RxXPI2A/M1x8ZS/A4rTr7Gx5h6qKIqAIBEXAtWRBpzQ1ApoR2ipMhxqGahdtRdId93ObF3/TJbf74CQJFUVAEWgaAVeSBVdabt26FfYIsenqhR9C7YIExNlRKt5HgOZLewYdTZpKFt5vc61h2xBwJVnQBEVp6dqK5qDiViE0RegWIM2h5J0wtrdtggrUNLxTS62JIhA5BFxJFtQASBTJyckRQ4KjTK674IpuOj1VvI0Ap0136tTJqmRbJ0l4GymtnSJgEGi7wT/GSHKnWM5iGT58eMRz5oruwsJCbNiwATRLqYSHQLVwq4P2bQ2v0BKra24euFanc9fuqHLh+KCdrENt78rhXthNpBEdhIDryILTZemQ5MaBkRZqKtwziludU8uwbdqRzsdr6ZWWA4/Md1+tkpCPTu0yMGte+O9BcUot00Spvv4gp5RGy5EICLiOLGiC4hbj0XphDXejpd+C8/DVPBHeT8Aii3nhxXVWLG4RY0xRzipX6NJkygbLShahcdIYkUPAVUosZyrt2LEjorOgAqGkk5uzrNTRHYiMnisCikAiI+AqsuAsKJqKuC4imkJ/Bfec4kuVVBQBRUARUATkjc5uAoFkwVlQ0fYlZGRkWD4R1S7c9HRoWRUBRSCaCLiGLLgQjzNXIrm2ojlgqV1w1pW+ZrE5lDRMEVAEEgUB15AFTULUKGJFFpyDz72CdAsQ5/8UThoCTOkX2XIe1As4o5nZ2f7hnMJ69QSgt24tFdlG0NQchYBryIImKPoqojULKlircPos89Wtq4Oh45xrUSELWWZzVnNk4ReeLGTxi/FKFs55IrQk0UDAFWTBbRm4F1SstAobaG4J0aFDB9UubED0OygC1XXAPg8A/10XNFgvKgKeQMAV6yzs9w5EYyFeqFakdrFy5Ur069cPKSkpoaJreJgInDoUOGIAwMVl7GT/vQColWXgXJF82xTgn3OBc0YCQ+TV6nM3AA/NASb3BU6X0X6x7CT/4hJgqdkizMqRpqCL9wMOkzgbdsoiQVn3sdLvfVbDuwE/Hg30ygKWlwL/kPRL/Hak7yMmpLNGAMNzga/WA9QW/KW5cMa9/XCT54qtwAWSzyr5zs8EjpQ6Vsrq8BcWA5/7vZDx0D7AcYOAnHTgjWVAF1kXuLEM+HCVf656rAg4BwH5aTpf6K/gFuLczyfWwoV5NH1xCxCVyCBw82HA7yeZDnVOEXDFWCGH40za7HhJEk+fDMhmsFiwUcIPMOfXHAjMkWbo10k65uMbl4X+hdOGAdNXAgNkZvVb5wB9JR6F/oXXzgI6Cte/sxzYL1/inQd0r3/BYid5rJ49VeKJaekj6ayP2ge4SIjHllDhJCqWmeRAmSREcMdU4/P4QgiC4U+fAgyq33RgSj/gsRMMMbI+vzsUuOFgYF/dJZ3wqTgUAVdoFnwxEU1C8RASBVd1cwsQzpCKpc8kHvWNdp79OwMXysj7l+8Db35vcntXOvDPLgQm9DTkwKtvS9hd/zXh7Pzpl5jwmBl98765lxmtY9kWE4ej9xNfMNrJs4tEW7nIkNBvPgZ+K53xzNXiV5A8Kc+LVvKOkMnPxwE3zQQuHQPsrgZOeckKxjNy/+tCLraECrfj+X8zvTNfNntmPbkQmH85cEhvo9XcImRJzej3M80dn6wWjeJH/nfrsSLgPAQcTxZ0LpeXl0dlL6hwm4P7RXFWFF+9yWOV1iPA0bMMtDFavmnysWVXlRlZU5OgfFNivvm3cDuwbLMhCp5zexFKrmgGNll8VmiIwoQYk88oySM12eRDkxNH77bU7fGN5Gmi+p+YwvyF5MLOnRIq3MRq/HeRlF+ysITfNDFlpMrmIqLF9BXC/OTT+kD5olls7Q7fuR4pAk5EwPFmKGoVfBtePF95SV8FX46ji/Ta/ghnS2dZI+alqloxM0kvan/+LaPv7+u1BOayraJxXmUyUrdF5jvsJfRT+Mvm3cYfwj2UaAbiSN/Oi9+z1gDvLTd3sAMP9FGwjLaECrfj+X8zP3+hP4bSSXwTlA1CHv6yPaC+/mF6rAg4AQHHaxacBdW5c+eor9oO1Rg0QfFdF9z1Nl4msVBldEP46m1Aioz26cidW2RKzM78dPE30CncWhnhp6UwDfoNVkp61EJ2Vhqn+J/rzVoMp4PZJoRFm4DD+/GqTw6u1yp4JVS4767QR+tFgyCRjJTy2g76bhkAy09/iYoi4FQEHK9ZcCZUtPeCCqdxOIWWu92qdhEOWk3HobmHM4aumWgcvpwN9Ss5/s0h0qmLKaq1coBYB0kAnE11zgjpjLsDT31jUqMPwpp91d9oGeMl7qPiYOZMJMqby4CCLOPUZnk4S4np2RIq3I4Xzjc1jMfnA78+2JRz/3zxzUwV05wQpooi4GQEHK1ZcIdZbvPhBLJgI1K7mDdvnrXtiP2WNSc3rhPLxtH8pW8C90wDZpwPlNcA34k/4uoPgK1iimFn3RrhLCdOueWMJJqpbpoJcCYS5W9fir9AZkJxxhU7601ionp4rpkZxfD54ie5boYQlnTgNwpp0YT1ynfAQHGshxNuYoX/9y9SHpIDF/KlyS/wP9+aBX100qsoAk5FIEkWvMnPx5lCpzJnIU2cKENPh8jChQutnW9HjhzpkBLFvxjfiRnn6OdaXo4s8SckiyYQ6J9oeUq+O0gWm8SZbfsIfCFG66DJh87mpiSvozFZtTa8qfv8r9Nxvlgws+tNpeKrSwCSCGdJhSP0xSy5MpyYGidcBDhFf/HixZg0aVLczd7hljmW8RxthqIJymkjeC7So9OdW5irtA0Bmp3sDrNtKfnuJhEEIwrGoFbTHFEwDhf8NSehwpu71w67agJw31EAF/rlCnn9n2g0HUTz4YwuFUXAqQg4liyo8FjvRxbntpOEW45wC3PdYNBJreKustzwIbBdnO5vnC1TfC8y/pEfvQYUNaPxuKuGWlovIuBYnwW3Bq+trbVmQjkNeGoXy5cvR//+/cE366koAi1BgNuQXPW+WW9CM5w9K6slaWhcRSDWCDhWs6Bzmx1xenr9lJVYI9NMflxzwbUf9KeoKAKtRYDOQiWK1qKn98UaAUeTRTwX4jXXEHyvBrUL7hdF7UdFEVAEFAGvI+BYsqC/wqlkwYeC237Qr8KFeiqKgCKgCHgdAUeSBfeDqqqqctxMKP+HITk5GT169LAW6Tl49rF/kfVYEVAEFIFWI+BIsqBzm6aerCxZVutgoSmKpMYtQFQUAUVAEfAyAo6cDUWy6Nixo+MXxvD9Gt27d7em0fI7UYVbiOsW27Ft/cCND2Obu+aWiAg4liycrlXYDwu3AJkzZ4712lenbEtily1W39yywn6xT6zyjEQ+9ItxCvTYsWMdPzCJRH01DUWgLQg40gxVVlbmeBOUDTo1IL7uVRfp2Yi455v7ju3aFWLJtnuqoyVVBKKKgOPIgs5t/oj5GlW3CH0X3EqdJKeiCCgCioAXEXAcWbDDpXObI3a3CM1PJDfVLtzSYlpORUARaCkCjiMLmgX47gi3veuavgvOiqqslE1/VBQBRUAR8BgCjiQLN5mg7OeBs6E4O0pfjmQjot+KgCLgJQQcSRbc1dWNQt8FV3TT56KiCCgCioCXEHAcWdDB7Vay4Ipu+lu4Z5SKcxGgqZObQPJlNxQes82U5J3bZlqy+CPgqHUWJIq6ujrXkgX9LNwzip0PfRgkDhXnIVBcXGxNRmD78LNy5Uprny/ucJyoa2Wc10paIqch4CjNwn77nBO3JQ+34WiK4gh140Z5sbOKIxHo1q2bVS7u6WV/uOV8Z4e9aMuR4GmhEhYBR5FFeXm59Q4LbtLnVklJSQHfd6GObue2IHcz9n9pFTXC3Nxc1QSd22RaMgcg4CiyoBmK02bdLjRBkfhsm7jb6+PF8ufn5zdMz6bpM5H39vJi+2qdIo+AkkXkMbXe7sd3desivSiAG6EkSQ4kCQq1QTVBRQhYTcazCDiKLLigjWsVvCDULrhRHV8Pq+I8BLhDgO0bo9lQRRFQBJpHwFFk4RUzFCGnXZwf1S72fgDFreyIf3n5hiRyu+c6ojxERUURcCoCjpk6y1kp1dXVntEs2ODULr799lvLf2GPYp36IMSyXCvqVsQyuybzqsuvQyf5V9KxBCV1JU3Gi1VARlIGCpIKYpWd5qMItAgBx5AFiYJC+7FXhFM06bDnzKhBgwZ5pVptrsfMupmolX9xF+rV7JuN6yLuxemb1BcFyUoWcW8ILUBQBBxjhuLrSSn+UxqDlthlF6ldcM2FTYYuK74WVxFQBBQBCwHHkYWXNAsiTOcpF3xxVbeKIqAIKAJuRcAxZMFVz1wc5eYFecEeAnsLEO49ZE/VDBZPrykCioAi4GQEHEUWXiMKu+F79uxpEQV3pFVRBBQBRcCNCDiGLGpray1zjRtBDFVmmqG4Yli3AAmFVHjhnzz0CVbNXhVe5DjGKt9RHsfcNWtFILIIOIosvKpZsMm4wSAXHfJteiptQ8Aii6+cTRbPX/U8Pnngk7ZVVO9WBByEgJJFjBqDU2i5WZ0u0osR4HHOxg2aT5wh0uxdhoBj1llwUZ7X3//AabRz587Ftm3bdC+iFvxQvp3xLea8NAcVOypw8CUHN7rz4wc+Rt7gPDDOjuIdOO53xyF/SD5W/HcFZj0yC9s2bEOP4T1w5DVHIqd3jnXvyi9X4rsPv8M+B+9jxWmf2h77n7o/9jtxv4a0mdaMv87AuoXrkJWXhQnnTcCIaSOs8KryKrx49Ys4+tdHI3efXOva1nVb8fYf38Zpd52GLx7/AltWb8GCtxYgKTkJx9xwTEO6eqAIuBUBx2gWnCnEmUNeFr5bnBvWqXYRfisv+WAJ/n7y3633TvQd2xdPXfoUSgtLGxJg+DNXPIPCuYWoLJO9xTLT8M3b3+CeyfegfHs5xpw6xiKOW/e7FZtWGhNgyQ8lFhH8+8J/o/+4/sjOz8Zj5z+G2S/MttLdtXUXbh93Oxa/txj7nrAv9tTtwUMnPYRP//mpFV5bVWsRAgnFlrItZda1qt1VyB+aj7SOaehS0MUiKjuOfisCbkZANYsYtx61i0WLFoGv9uRmdirNI/DSNS/h6BuOxvG/P96KOPaMsbhp2E2NbkpJT8E1H16DdslmsHHPYfdg3DnjcPGTF1vxJv1kEn478Ld486Y3cckzl1jXKnZW4JJnL8GoY0ZZ57yX2sL4s8fj/bveB8Nv/+F2UOuY8rMp6NyzM1678TUceMGBjfIOdjLq2FF469a3QHLz11aCxdVrioBbEHDMUJ5mqESQnJwciyRUuwjd2pW7KlGyvARDpwxtiNytfzfL7NRwQQ76junbQBTUCrYUbsHIY0b6R8Go40ZZ2od9kSQw5LAh9imGHzkcu7bsQunaUqyZtwbDpg6ziMKOsO9x+1oEUrys2L6k34pAQiHgGLIg6l73WdhPFrULzoqytzixr+t3YwQ4uucggtOq/SU5pfGbFDt29Wlo5dvMdFWagPwlu3s26mp9m0Bl5mYiNSO1IUpGlwzrmKYsTnmlJuEv2XnZ1ql/Gv4DnLpqX9r+9+mxIuAVBBxFFl4BNVQ9+OIdbmui6y6aR6pTfiewk/5uxncNEXdu2ol136xrOA88yOmTA5LJkulLGgUtmbEEvUb3ari2bf02FC/3aQlLP1pqkUfekDx036c76AvxF57TVFUwsgDJqYasqPnYsrlws32o34qAJxFQsohDs1KD4roLrugOHDXHoTiOzpI+gtnPz8ayT5eBJqa3bnmr2fKyQz/0skPx1bNfYdF7i8CZS7MenYVVX67C2NPHNrr3ndvewY6SHVglaza+eOILyx/BSRaTLp+ETSs2Yfo90y3T0/LPlluzpujsTklLQWp6qqV5cNYTtR86zt/703uN0s7MyUTR0iJsL9re6LqeKAJuRcAxDm63Atjacvfo0QOFhYXgnlE0S6kER+CkP56Ess1lePCEB7Gndg+GHTkMvfdvHq+Tbz8ZnJXEWVTJ7ZNBk9PZ952NcWeNa8ikQ3YH1FTW4Mb+NyKpXRLGnDYGZ/71TCt88GGD8aN//Qiv/PoVvHnzm2jXvh1GnzgaP37kxw33n/vQufj3Rf/Gr7r9CunZ6Tjr3rPwxIVPNITvd9J+ePFXL2L5rOW4c/WdDdf1QBFwKwJJYnd1hGd56dKl1ih7xAgzl92tgLak3CtXrkRJSQkmTJiQMP4a4vNozaMtfp9FdUW1NTU2s1tm2BBXV1ajbFMZuvRq7L/48pkv8Z9r/4O/FP8FnPJK3wW1hUDhT4PrJ2gOC/STMC6ne9OcRf9GsGnfNVU1qKupa+QbCczD/5zvszg6+Wj/S3ocQwS2bNmCxYsXY9KkSQn1ewwXYjVDhYtUFOLRFMX3XJAwVJpHIKVDClpCFEyNJqNAogjMJbOrOLqDEAXj0VzIhXzBiILhJAiGByMKhnPGlb8TnddUFAG3IqBkEceW44ue6OzWabSxbQSLeMQ0paIIKALhI6BkET5WUYlJ7YIL9EpLfauSo5KRJtqAAB3dtyy+peFcDxQBRSA0AkoWoTGKagyu4uZCPdUuogqzJq4IKAJtREDJoo0ARuJ2zobi5oI7d+6MRHKahiKgCCgCEUdAySLikLY8QW4umJWVpdpFy6HTOxQBRSBGCChZxAjoUNlQu9i8eTMqKipCRdVwRUARUARijoCSRcwhD54hX4yUlpamW4AEh0evKgKKQJwR0BXccW4A/+ypXXChXr9+/Tz7PnLW99B2h2KP/FNpjEAmdDpvY0T0zEkIKFk4qDXy8/OxevVqrF+/Hn379nVQySJXFJLEoHaDIpdgG1IqKyvD+nXrMXjIYMes2CU+SfJPRRFwGgJKFg5qEa4ELigoaNgvqqmVwQ4qcouLwo7QKZ0htwMpKZb3ZQwZ6pgytRhQvUERiBEC6rOIEdDhZtOzZ0/U1NSguNi3fXa492o8RUARUASihYCSRbSQbWW6fM8FzVG6SK+VAOptioAiEBUElCyiAmvbEuUWIJxCy6m0KoqAIqAIOAEBJQsntEJAGdLT09G1a1fVLgJw0VNFQBGIHwJKFvHDvtmcOY12x44d2L5d37TWLFAaqAgoAjFBQMkiJjC3PJPs7Gx06tRJF+m1HLqQd3ACAUmYu/1SeExiVlEEFIGmEdCps01jE/cQahdLlizB7t27kZGREffyeKUAfJ3tunXrGqqzcOFC63jUqFHWDsANAXqgCCgCDQioZtEAhfMO6Leg/8K/Y3NeKd1Xoi5dGr9mlTXgW/GoyakoAopAcASULILj4pir1C645oKvX1WJDAIki+Tk5IbESBQkZv9rDYF6oAgoAhYCShYOfxDy8vKsfaJUu4hcQ5EciCu/KXv27LHOI5eDpqQIeA8BJQuHtyk7NK7q3rBhA+rq6hxeWvcUj+8+J0lQqFFQs1BRBBSBphFQsmgaG8eEcL8odmxFRUWOKZPbC0L/BFfLU7g9vK1luL1eWn5FIFoIKFlEC9kIptu+fXv06NFDHd0RxJRJcVsVCrUMFUVAEWgeAZ062zw+jgnlFiDcurykpMS5nVtNObDLPRsg9sisRVWXjujSThY+umnxY6d+jnkutSCJg4CShUvamm/Ro7mEGww6diS8/nPg5WkuQRRIl5IOdU1p/Qp6TY3M9fXN5vIL0UNFIGoIqBkqatBGPmFOo+ULe7Zu3Rr5xDVFRUARUASaQUDJohlwnBaUmZkJrhHQabROaxktjyLgfQSULFzWxtQuSktLG/Y1clnxtbiKgCLgUgSULFzWcNQsqGHoy5Fc1nBaXEXA5QgoWbiwAaldcFZUZWWlC0uvRVYEFAE3IqBk4cJW46wozo5S34Wv8aprgVumA4WlvmuROHpQJnjNXtN0SqHCm77ThHy8HHhidqhYGq4IxB8BJYv4t0GLS2BvAcIV3bW10kuqgGTxByGL1RGeKPaAkMWXhU0DHCq86TtNyMc/AI8rWYSCScMdgICShQMaoTVF4Ipukgb3jFJRBBQBRSDaCOiivGgjHKX0ufkd94yiKYqru924t9GOCuDPnwBz1sr+TJnAheOAqYMMYBzNfyKj7sn7AA9/CZSJe+aCA4BjZBXd3TOB/642YReNB3L83gtVUgb87FVg5RZgYl/g11Nk8Z3ZAspK+Kk5wNvfAhWyrm2KpP2LQ4D2fuvbpi8DXlwA7JD8Lp1gyuL/N1R4c3ViOizXY18BC4TjD5P8a3VvSH949djBCKhm4eDGCVU07kbLV4TyfRduk13SGY/9G/DeUuCkkdJhy5N43KPA09KZU5ZvAv7yKXD5y8B+BUCKdOinPQkc/YjpaI8aAtwvJqK7Pjbx7b8Xvyg7d8iuI0cOBh4RkmF8W375OnDtm8CgbsBB/YSoZgKnP2WHAu9LWU58XLYsl0sH9AKYVqGfWStUeKg6le4GjngYoOnp+OHAa4tMHXwl0CNFwLkIqGbh3LYJWbLU1FRr6w9Oo7U3xQt5k0MisKMv2ilaxdVAp3TgyoOAwbnA9W8D5481hdwinev0y4Ex0nHTJ/HGYmB3NfDhj0w4O/W/fyGEcbyvUmeOlpH7Web84H6iXTwg8b8H+sjL8R6UuM+cC5yzvwk/fV8hjjuBT1eYUf7VbwA3TgVummbCmdbgu8wx/4YKD1Wnv30GZKaKVvQLvpnP1Hni/b709UgRcDICShZObp0wysZptHPmzMGWLVtc9U4GzjDKzwLu8NMM1st+fsViRlon3xRqG6MLzDE1i96dRVMQjcKW3I6GcOxzfh87zHd2QG+gcwdg7jqA5im+vuLrtcDCDb447LxpBqMmsXwzcHi9GYwx+nc1BMZjag3NhTNOqDotWG9MZ/XvXOItVn0+Wm4d6h9FwNEIyM9Rxc0IZGRkICcnx3WL9LaKqSgjBWgnI2z7QzL4zeHmnG3SSTr6ZL8nlJ1stlyzxb/Tta8xDVt4b1chlMoaYJvkR/JJE9Kx8+M3fRYjZKfynUIGdUImNaLB+EtKff6hwnlPqDqxDLVUh/yEJKiiCLgBAdUs3NBKIcpI7WLBggXYuXMnsrJkuO4CGdhNTEzLgNuOls67vkNeISP7L1aLs1s6+NYKtYbxfczdq8SZvEI+g8S81VWc4NV1wAkjjL+CMehcfnKO0R7ys42mM11MVpMHmvs3iTbyTf37pkKF845QdaI57d3vTNr2X9UqbCT02+kI1P9MnV5MLV9zCPCtb9nZ2a7SLn4y0ZibbpkhI3LxTRTtAM57zsxUSm3DEIZ+iY2SFj+/ex/o1Ukc6EIQU4QAhghp3CTXlmyU2VDi++C6jP9726etXHgA8Nx8YOYPpkw3f2Cc3Tb2ocJD1Ym+kjXbgPtmAeWS/0sLDDna6eu3IuBkBNrws3RytRKvbNQuvv32W1RUVKBDBz9bjUOhGCej/2fPM05jzmhKkyfxiMHAA6e0rcDHic9i4J3GId6vC/DmxWLuEr8E5Y2LgIteBEbdY67t20NmX4nDu1u9JnP7McDmXTIr6zGjdUwT/8iYnuZe/g0VHqpOnMr7xFlCUO/IlF4hqbxM4MfizF9a4stDjxQBpyKQJO92DrCixqeoS5cutVYjjxghw0CVViEwe/ZsawvzQYP8vLStSqmVNxXOaNXLj+jYpumpLRqFf4mrxEexSTr9nqJVBBNOra0RExT9GcGEWkdZlY9EAuOECmf8UHVieIGYvoL5XQLz2+tcX360FySRuMBJIosXL8akSZNcuW4pEhg0l4aaoZpDx2VhXJy3ceNGVFdLb+ciYaceKaJgtZlWU0TBcE7VbYooGN5BHO+2tsHzQAkVzvih6sTwVhFFYGH0XBGIEQJKFjECOhbZcK0FV3brFiCxQFvzUAQSCwElCw+1dzuZVsRV3evXr0ddndhZVBQBRUARiBACShYRAtIpyZAsSBQ0R6koAoqAIhApBJQsIoWkQ9Jp37498vLy9F0XDmkPLYYi4BUElCy80pJ+9eA0Wk6h3bRpk99VPVQEFAFFoPUIKFm0HjvH3sl1Ft26dXPVIj3HgqkFUwQUAQsBXZTn0QeB2sW8efOwfft2cIV3TCRL9rMY/dOYZJXYmcimViqKQIwRULKIMeCxyo57RHXu3NnSLmJGFp0Gyr4af4taFbl+5IcVq0Rr6opc+cRK+M6QlasKMaB/X9AnFHfZIzPdktQoEPd2SLACOODJTzDEY1hdahdckbp7925wd9qoS7KsZouS0P+yfPlyax1Jj56iwSTX7+ERpfwaJbunHUq3bUftykIMGyb7iagoAgmIgA5PPNzo3LqcJMGXI7lVqE1wz6vvvvsOubm5OOCAAyyNKZb1oTYxZMgQa8JASYlu5BRL7DUv5yCgZOGctohKSbgFCDu4qirZ7MhlwnJ//fXXKCsrw7777gvuecUV6vGQLl26WAseqd1UVsrLL1QUgQRDQMnC4w3ONRcpKSmuWndBYqP5jJtLdu/ePS7aRLDHYsCAAUhLS7PKFSxcrykCXkZAycLLrSt1S5Ld6riqu6ioyNrV1+nV5cpzahPl5eUYPXo0Bg4cKC9HcsZjSiyHDh2KHTt2WFuqOB1LLZ8iEEkEnPErjGSNNK29ECgoKLCukTCcKjTtLFq0CN9//z169OiBsWPHxm7KbwtAyczMRJ8+fbBq1Spr4WMLbtWoioCrEVCycHXzhVd42vnZAa9btw67du2yZhU5aXU3SYzaBAlj//33B809TtEmgiFMskhPT8eyZfJeWBVFIEEQULJIkIbmugu+52rOnDnWFuabN8sLr+Ms3JJk4cKFFnnRVEZtwg3vEKc5irOj1BwV5wdIs48pArrOIqZwxyezb775Blu3bm309i921LEU+iIKCwstQuBUVG6jTlMOR+hjxowBzTtuEpaX61hYB26tQse3iiLgZQSULLzcuvV1Y4e8bds2S7OwqxvLqbSc+kpfBIUznLgieufOnZbtnyYdjtTdKH379rXWXnA67ciRI91YBS2zIhA2AmqGChsq90bk+gQuaPPvlGNFFrW1tdY0WKJHMxjfc8yFdtQm2Nn6l8ltCLPsgwcPRmlpqe7w67bG0/K2GAElixZD5s4buE1F165dGzpnviCJnXe0hSuvSUx2XuxgSRbcGdcLwn23+DrbH374wRVTk72AudYhPggoWcQH97jkOnz4cHAlsi3R1i44+4qahE0UzNcmCydP47XxCfebs7co9F+oKAJeRUDJwqstG6Re7KhpW+dutJRokgVnCq1YscLKh/lSqE1wRTm1HHvthxXg8j902JMwNmzYYE1Ndnl1tPiKQFAE1MEdFBbvXmTHPWrUKKxZs8aaiRStmtKhTnKgJkNy4ic1teU7xdJSVs810SpqRNIlCVJborN7v/32i0iamogi4CQElCwc0BqjHwYqa2JZECqU/aKcYR9Jn5/Wy+H9gb8f2/r7Y30ntybhC6e4ASL3tFJRBLyEgJKFA1qzvFrIotYBBXFYESpiSqBtrzzXXnClPM1vnEwQrx1y214TTUER2BsB9VnsjYleUQRajUD//v3BmWZufodIqyuvN3oaASULTzevVi7WCNDZzfUjnAkWzQkEsa6X5qcIKFnoM6AIRBgB7nNFZ75OpY0wsJpcXBFQsogr/Jq5FxHgjDOao4qLi623/HmxjlqnxENAySLx2jxojTMDZrUGnge9SS82iQC3V+EOuqpdNAmRBrgMASULlzVYNIp762TgotG+lKf0A5462XeuR61DgNoF943avn176xLQuxQBByGgZOGgxohXUfbPb5zzkK5Ahk6qbgxKK87sxYirV69uxd16iyLgLAS0S3BWe0SlNKnJwMWyqHjfPCBLzE0rtgKPzAPW7wQuGwP0ygam7QPUymrpufLm1WMGAj2ygDunAnd8Dpw8VPY9knvy5ZUTRw4wa0JeWAx8vjYqxfVUotQu5s+fb71PxH9fLk9VUiuTEAioZpEAzfy0mJROGAx8IZ37J6uBg3sDL5wGcMemFaXAblkUuLEMWC7HW3YDRXLMhYILN8r+UbJYcJIsxL5DiOOM4SaNdnLj06cAg3ISALw2VjE7Oxs5OTlQ7aKNQOrtcUdANYu4N0F0C9BZdgLfUg789mPgB9EOKCvl+0khkK4ZwMergV9J+DfFwIyVVjAWCEn06wQ8v8Sc8y8J5cyX5Z0UcvzkQmD+5cAhQjokGJXmEejXr5+1DQj3y7I3cWz+Dg1VBJyHgJKF89okoiXaJm9P/em7wPBuwOnDgH1kh/LxPU0WHZLDz2pRiSEK3kHCoCaSETCDKvzUEismZ0XRBMXNG5UsEqvtvVRbNUN5qTWD1CVNCIEzm/5zBnDiEDEvyX5Lry8LEjHEJWoW/kL/hkr4CPD1sXwPOl8nq6IIuBEB1Szc2GotKDMd1/RRHPKE8UXw1mnipKbQ9xBMYvACvWDZevoaNQr6L6hdjBgxwtN11cp5EwHVLLzZrg212rQLSBZS6Cb+CUpPmeV0/UHmOK1+qLBVTFUDxVndvT7Otko57gj072zuNbH1b1sR6N27t/XmwIoKAVxFEXAZAkoWLmuwlhb3y/XAi+Kofv5U4OtLgZfFHHXfV8B2IYQRuSa1D1YAxw4C3jzbnH8l99DMNPMCM922pXlq/OAIdOvWDWlpaVi/XgBWUQRchkCSvB/ZEdbnpUuXWi+8T0QVffCD0X+fRYoMC7qkAyWiaQQThifLx/8dEtniwN5RFSx2bK5N7Q88fmJs8opVLtyNtrCwEBMnTtT3XcQK9DDz4fviFy9ejEmTJlnvig/ztoSJpppFgjR1dV3TREEIGO5PFLwWT6Jg/l4UvhyJ47ONG2V+sooi4CIElCxc1FhaVPcjwLfn8X3dGzZscH9ltAYJhYCSRUI1t1bWCQgUFBRg9+7dusGgExpDyxA2AkoWYUOlERWByCDQsWNHa/vyoiLZiEtFEXAJAkoWLmkoLaa3EKDvYtOmTdakDm/VTGvjVQSULLzaslovRyPQvXt3a8YN36anogi4AQElCze0kpbRcwjQ0c11FyUlsumWiiLgAgR0uw8HNBLfM1EtW4GrNEZggGx66GWhdsF5/ZWVldZiPS/XVevmfgSULBzQhlxV7QSpqanBrl270KmT7E/uEKmTJaNN7WHlkCK2uhjcibZ9+/aWdsGtQFQUAScjoGYoJ7dOjMvGd0UvWLDAWjQW46ybzM6rRMEKJyUlWaYoOrpVFAGnI6Bk4fQW0vJ5GgGaorhtuW4u6Olm9kTllCw80YxaCbciQJMfTVHcl0hFEXAyAkoWTm4dLZvnEaApir6L0lJ9P63nG9vlFVSycHkDavHdjwCn0PL93LW1OiXO/a3p3RooWXi3bbVmLkEgJyfHmlTA166qKAJORUDJwqkto+VKGATos8jMzLS0i4SptFbUdQjoOgvXNVnkC8ztsteuXYu6ujrrhTyzZ8+2pnUOGzbM2vAu8jlqioEI0NFNU5SKIuBUBJQsnNoyMSwXX8bjP3XTtp1zSwqV2CDQuXNn63WrXBhJTUNFEXAaAmqGclqLxKE8nOsfKBkZGeBHJTYIkCwoql3EBm/NpeUIKFm0HDPP3ZGSkgJ2VpzGSeF3fn6+5+rp5ApRiyM5c4GeiiLgRASULJzYKnEoE1/1SXMUhd/BtI04FCuhsszKykJZWVlC1Vkr6x4ElCzc01ZRLWlubm6DZpGdna27oEYV7eCJc0aUahbBsdGr8UdAySL+beCIEtAMwvn+FGoZKrFHgJpFdXW1tWV57HPXHBWB5hFQsmgen2ZD61DXbLjbAumnoL+CWoZK7BHgu7kpaoqKPfaaY2gEdI5eaIyajiEm/oV7FqKwrrDpOG4KkQk5SeOT8F7Se0CNmwredFkPaHcACtoVNB3BQSHU7tLS0lBeXu6gUmlRFAGDgJJFG5+EbXu2oUj+eUY89kSUw10db3p6upKFZ35M3qqImqG81Z5aG5cjoGTh8gb0cPGVLDzcuFo19yFAsvBfTe++GmiJvYqAkoVXW1br5UoEUlNTUVVV5cqya6G9jYCShbfbV2vnMgS4mp57c9n7c7ms+FpcDyOgZOGwxi3f0dghG3ge7+LOf30+nr78abz++9etojitfPHGp635U7OgcL2FiiLgJASULBzUGoveW4T7j7u/oUSB5w0BcToonFuIh894GCSIzj06w2nlixMsEc3WJgs1RUUUVk0sAgh4bKJkBBCJYxIbFm9A1S6fvTrwPI5Fs7LesGQDOmR1wKXPXYp27drhg7s/aFTeeJfPC/kTVwrfLaKiCDgJASWLGLfGN29/A5pySgtL0blnZ4w7exxGHj0S33/6Pea/Nh9b123F0z95GqNPHN3o/NQ7T0XxsmIs/XgpRkwbgU8f/hTbi7Zj6OFDMfWXU9EuOTwlcc38NfjkwU9QuqYU3QZ0wyGXHIL+4/s3oLDivysw65FZ2LZhG3oM74EjrzkSOb1zrLJ88cQXqKutw7NXPove+/Xeq3yzn5uNvMF52LxqMxa9u8iq37RrpyGpXRJm/GUGdpbsxNgzx2LsaWMb8tu0cpNVno1LNyI1IxUDJg7A4Vcdjvap7bH4/cVY8MYCnPzHk5HZLdO658N7P7QI6tjfHtuQhpcOlCy81Jreqkt4PYy36hy32sz8+0w8dv5jyO2fi4k/noiq3VV48IQHsfrr1cjqnmV1ruww+43rh049OjU6Z+dZ8kMJPrr/Izx+4ePo0rsL+k/oj9d/9zre/dO7YdVpR/EO/HXqX9G+Q3sccukh1tYed0+6G+yoKSSyeybfg/Lt5Rhz6hiQOG7d71awQ+/Sqwu69u1qdeIsX7f+3fYq37fTv8VTlz1lpTNs6jCs+GIF/n7y3/HP0/6JDtkdkDc0D4+e8yhozqKQVP64/x+xq3QXJpw/Ad0Hdcc7t7+DN256wwofeMhAixxfvPpF65zle/WGVzFkyhDr3It/lCy82KreqJNqFjFsx52bduL0u0/HoZcdauU6/pzxuC7/Oqz6ahWm/HyKNcLftGITDr3UhHPE73/Om3Zt2YVrZlyDnqN6WmlQA/juo+9w/O+Pt86b+0MzEgnqhJtPQHb3bBxw5gHIH5rfsDU5O+Vx54zDxU9ebCUz6SeT8NuBv8WbN72JS565BIMOHYRlM5c1lG/9ovV7lS+tYxqufPVKJLdPtgiNPg7W+Yirj7DSXP7Zcix8ayH6ju2L4u+LrTKc//D5llmLEYgR8aB0yOyAS566BHcfdjeGTh2K1258zSr7PgftY4V79Q8JQ81QXm1d99ZLySKGbcdOmqYjmps4ml/3zTpUl1ejuiL8mS8pHVIaiIJFp4mImkk40m98P+Tuk4ubht6E4UcOx4ijR2DijyYis2smdm3dhS2FW3DSbSc1SmrUcaNAjSFc6bVvL4soGD93gNmQcMRRIxpupzlpx8Yd1jmvDz5sMJZ9sgwbv9uIDd9twNKPliI7P7shPrWnY2881pqBRY3iqP87qiFMDxQBRSB2CChZxA5rfHTfR9bouGBUAQYeNBBjThuDlV+ubFEJaKbyF/oDYN5Z5H856DFH6jf87wZ89exXWPzuYjz/8+fxn2v/Y2kCNDFRuhR0aXQvNRD6KcKVjl3Nzqn+8WmCCiYky/uPvR/JqckYdMggUGOoLKu0NA7/+Omd063T2ppa/8t6rAgoAjFEQH0WMQK7qrwKr/7mVZx212m48csbceZfz8T+p+xvOX331Jne3n6tqV2kwHP7emu/afah3X/ylZPx87d+jruL7kb+sHx88tAnyOmTg+SUZCyZvqRR8ktmLEGv0b0aXbNP2lq+t299G/lD8nH78ttx8VMX47ArDgOx8Ccnms5ofjr7vrNR9G0Rpt893c5evxUBRSCGCChZxAhs2vA56t5evN2yR9N3QB9BTVVNgxkqIyfDMlOxU+coOvA8EkV98uInMfeVuVaHXLGjAru37rZMU5xNRV8KtQ6unyC5zXp0FlZ9uQpjT/fNXvIvQ1vLR3MTzV/Mi69yXfDmAsx7ZR5qKs3+6MTm8R8/jn2P3xeTfzoZZ917Ft78w5vgjC4VRUARiC0CShYxwpuj9tPuPA1zXpyD63tcj+t7Xo+0zDTLwbt2wVqrFHQgs9O+ecTNKJxTaOh+bPwAAEAASURBVDmU/c/bWlROaz3tz6dZDuurc662nNc9R/a0fAJM++TbT8awI4ZZM5iu6XYN3rntHWtEP+6scUGzDixv0EjNXKTTO71TOq4vuB7X5l1rTaGl5kWnPhf+cVYUHfjnPHCOlcr4s8dj1LGjLAIhwagoAopA7BBIkhFdmBbv6BZq6dKl1n44I0b4nKHRzbHtqdftqcOsullYumdpixLjWgqOqqltBJPd23Yjo3NGQ1DgeUNAGw62b9xu5UGHeaBUV1ajbFOZNV02MCzYeVvLV7alzJqSywV/kZYj2h2Bfdq5a/bUrFmzMGTIEHTv3j3ScGh6zSCwZcsWLF68GJMmTWp4H30z0RMuSB3ccWhyrlloTvyJgvECz4Pdy6mWtu8jWDiv+ZNTp/xOTUVDSlpK2ETBRMIpX5OZSQBnY6koAoqAsxFQsnB2+4RdOjqLP3v4sybj05z153V/bjJcAxQBRUARaA4BJYvm0HFR2Il/OBH8qCgCioAiEA0E1MEdDVQ1TUVAEVAEPIaAkoXHGlSrowgoAopANBBQsogGqpqmIqAIKAIeQ0DJwmMNqtVRBBQBRSAaCKiDu42otkM7JMk/FWcioG3jzHbRUrkPASWLtrSZcMShyYeC/7wiXK9hv1PBK3XSeigCikDbEVAzVBswpFbhJeEKVq4edsiifi9Bq3VRBFyPgLd6O9c3h1ZAEVAEFAFnIqBk4cx20VIpAoqAIuAoBJQsHNUcWhhFQBFQBJyJgJKFM9tFS6UIKAKKgKMQULJwVHNoYRQBRUARcCYCShbObBctlSKgCCgCjkJA11k4qjniUxhOmd20aROqqqqQkpICvoiK79fu168fOnSI/AuJ4lNLzVURUATagoCSRVvQ88i9ZWVlKC4ubng7WElJiVWzgoICJQuPtLFWQxFoKwJqhmorgh643359Jxfj2QvyUlNTkZ2d7YHaaRUUAUUgEggoWUQCRZenkZ6ejo4dOzbUgiao/Pz8hnM9UAQUAUVAyUKfAQsBkgNJgkLtwtY2rAv6RxFQBBIeASWLhH8EDAC5ubkNJqhATUMhUgQUAUVAyUKfAQuBtLQ0dOrUyTpWE5Q+FIqAIhCIgM6GCkSkped11WK3qWvpXY6Mn5ebg+3bt6N7VyGN2kpHlrHFhUpOlVv0fSMtxk1vUAQCEFCyCACkxadJAuHSZ4HieS2+1Wk35MuW6xnogg6z33Ba0VpXnqFnA3ljgHb6mLcOQL1LEfAhoL8iHxatP1r1rhDG862/3yF3cvxtDFEOKVBbi5E7ypBFW9PR+xUBRcBjb+/RBlUEFAFFQBGICgLq4I4KrJqoIqAIKALeQkDJwlvtqbVRBBQBRSAqCChZRAVWTVQRUAQUAW8hoGThwPbcUdG4UIHnjUPjd/bCfODtb8PL/5sNwN2fhI7r1LqGLrnGUAS8jYCShcPa993vgKMf8RUq8NwXEv+jlxYCLF84slDI4q4QZPHzV4H7Z4WTmsZRBBSBWCOgU2djjXiI/BZvBHZV+SIFnvtC4n/06oWRLcNXa4CTRkQ2TU1NEVAEIoOAkkVkcAw7lcoa4D4ZPX+9FqDJZUgucO1hQN8c4NMVwCvfAOu2AZe9ZDpO//M/Hw88K2v/Bsk967cDby4BOkgLXjoBOGJw6CKs2Qr8cQZw53FA1/pNZu/8GMhIAa461NzPOLd9KCP8kyVtub5gPfDQF0ChXB+eB/x6ClDQycTl9U7ybqTzx5rzlVuAf38NzFkHjOstZRoEzPgeuPVoX9nmShi1h9LdwFQJ/6Xky/0L/zITWF0KvCF1ShZ99zdTgfnrTdw1gscAweeyicD4Pr609EgRUARih4CaoWKHtZXTUf8CaOufOhA4Zijw8Q/A4f8E6mTHkO6ZQK/O0nnLDhXsFHtkNz5PTQY+WAb85GXTKbOzrd0DHCVmqyWikYSSAknvZSEjduAUEtet04HbP+JOs+baa4uBedJJkyg+Xg4c+ABQVgWcMRrgyH/fvwAbhKgo06Usn68yx1t2SZ2kHkz7xOGG+E54HPiPmKps2S7keM4zwKgewDAhnhtlLaNtmhraHeiYBvQUIiIpFe8EJv/dlOMyIUMSysEPytpH814mO0n9VgQUgRgh0D5G+Wg2ggA7VBLCP04znSVBoWZx7GPAJgljBzpBSOKHzWYUzfDAc17rKGQy80rZxUKo/mcHSZp/AD6Sjn1EPkOblvZCNscOM4Rz9v6moyc5UZsg2YyUTpwO65NHmDSue8sQ2vPnm3OO7Mf8FfiTkMuDpzbO556ZwO5q4L+/MB37FVKuEXcbErRj1gghviBpjellrlBbmfkDcMPhwHFCMH+YDhwgYSeNNPVherdMk/plAWftJ/gIodikZqep34qAIhAbBJQsYoOzlQtNPy/92Jh2aK5ZJqPkz1aaApRLxxiusEMlUVD4zdE4R//hyIlCBNe8aWJSM6B2M2+d0XD6djEawd9ONFrHwiKj3dzwji9lmohoZgoUXmNa9a/EsIKPF2KiqcwWmsz2K7DPhDR6As/M8537H5Ek9+kqJrc7gWlDTNo/PsBnPvOPq8eKgCIQfQTqu5zoZ6Q5ABVCCEf/C5gk5hWaomhuOm9My5GhZuEvydzUKUxhh76pDFgsRECTEf0KNGdRM5ku532EMKhh0J9SJ6apTDENtZP07c+Rg4HTRu2dGbWm3qKl+EvndP8zoxHZJMcQptmUMN/ZvwTuOFY0lirgp6+I3+JPwCeiiagoAopA7BGQsZ5KrBB4XfwBH0lnt/JGX8fKaxR2zJTA/jPw3MRq/d/sDuIL2Ad4XsiKM60Ok+MuGcC9swCGnSwmIEqumMuypcOmn+NP0mHbQm0kRcxZgUITGDUUf6F/pbXy/SYzCeCnBwtRyIfkNe1fYv76ApgysLWp6n2KgCLQWgRUs2gtcq24L19s77Vit6fzllJYCvz2PXNMrYOSIx130Q6AnWVN7d7nJlbb/tIUdd/nwFgxZ5EgaPJhuV5Y4CML5nDlQcDTc40pieGfrRB/whPAZtEiAuVXk4wv5FoxcXEB3s0fAF+sDozV/HlXqft3Jab+jHnhC+KQFwc58yZZbN1tTFPNp6KhioAiEA0ElCyigWoTaU6WEfHF482soR63AIc+BNx0JNBZOuz50sFSJg0wU0eH3GV8A4HnJlbb/pIsuJbDnm5LTYEaBonqoL6+tG+eBpwtjuXTnwSyRBu6QDrv6yebmVG+WOaITus3Lza+j0OkXl8Vmim9nFUVrlCr4Wyt8fcBg3OBe04Afve+TM/9HdBfTFCcRfW7I8JNTeMpAopAJBFI2iMSyQRbm9bSpUtRW1uLESOkJ3OTEL53z2vR+yyqaszo3F6vEKy628qFRPxs/oHnwe6J1rVq0XCoDXHmVFPCFdpp7QFOgbXlqtfEiS8a0geX21dCfxMbzpqiP8eWjaJpEYuWEI9171GPyzzcH4lzRArmIpk1axaGDBmC7t39wHRR+d1a1C1btmDx4sWYNGmSTNSItAHYraj4yq2ahQ+LmB2lSt/VHFGwIP5EEew8WGFprqHpqqkP13K0Rqh5NEcUTHPWSlmTcb+YnlbJzKxKMwX30a/MlNeW5Els/ImC9+aL36TFRNGSTDWuIqAIhETAXUOukNVJ7Ahj/yYL5mQU3pScMgp4+PSmQtt2/YoDgbXbgMtfNgvnBneTxX7HGLNb21LWuxUBRcAJCChZOKEVIlSGBddGKKFWJMMFf3cdbz5cGU6TlIoioAh4BwE1Q3mnLR1TEyUKxzSFFkQRiBgCShYRg1ITUgQUAUXAuwgoWXi3bbVmioAioAhEDAEli4hBqQk5EYHy8nJUV9eveHRiAbVMioBLEFA3pEsaSovZOgRWrFyFLaVbkZaWhuzsbGRmZiIrK8v6tG+vj3/rUNW7EhEB/bW0udVlUd5h98gig5vanJImEGEEOvbAsHbp2LlrN3bu3Gl9ioqKsGqVLAYRSU9PbyAOm0Da+e90GOHiaHKKgJsRULJoa+sliSUvs0BS4UfFaQjIjF507pwqH9/yc5qlbPLg99q1a1FVVWWt2s3IyGhEINREdDWv01pVyxMPBJQs4oG65hlXBFJSUpCTk2N97IJUVlY2IpDNmzejpqZG3hfSDh07dmxEIDxXUQQSDQEli0Rrca1vUATo0+CnWzdZel4vdI77ayDFxcXW/mXJycmNfB80YdGkpaIIeBkBJQsvt67WrU0IkAD48d/Qb9euXQ0Esn37dmzYsEHen14HOsttv4f9TfJRUQS8goCShVdaUusREwRoguInP1/e9iTCTZvLysqsz44dO8CdS+kD4fXU1NS9CIQmMBVFwI0IKFm4sdW0zI5BgM5vW5Po0UNeuCFCTYMEQhMWCaSkpASFhYUWgegUXsc0nRakhQgoWbQQMI2uCIRCgE5xrungp2fPnlZ0vqvF3//R3BTeUOlruCIQDwT05UfxQF3zVAQEgcApvCQTTuGlUAPp0qVLg9aiU3gtWKLyZ926dVizZo2l+dF8SLKnxsgXsZHwVQwCqlnok6AIxAmBpqbwfvXVV9ZsK07n1Sm80W8ckoP/ljDUAinqX2qMvZJFYzz0TBGIKwLUKDiq5QwsexaWTuGNbpPk5uZi+fLlDZkQfy7O1OnQDZBYB0oWjfHQM0XAcQjoFN7oNgk1CJr8tm3bZpmimJs92y26ObsrdSULd7WXllYRsBBoagqv7UTXKbwte1Dy8vIssuBd9FvYWl3LUvF2bCULb7ev1i5BEPCfwmtXOXAKL1egB07h5bRfeyfeRN6Flyv3ly1bZkFHpzbXyKg0RkDJojEeeqYIeAaBcKbwcgV6RUWFVWeau+w1I/Y300gE4RYuXbt2tSYUqAkqeIsrWQTHRa8qAp5EgJ0id+B1yi68YvGB/HeE5OX3QGlpKbp2y0WdQwolvnbIf0eIkoUjmkELoQjED4GmpvDa/g9+t2UKr51OQcHe2/izM3xLrD+1juicc4BuB+OdFc7QpvIz5TU5veL3XATmrGQRiIieKwKKgLUokNN4I7ELLxe9ccsTfg8ePLiRVkOor5sBVJqlDQ5A3hlEQSCm9leycMADoUVQBBSBliLQ2im8nJJKoW9k4cKF4LqGgQMHqhO5pQ0Q5/iqWcS5ATR7RcDNCIQzhdfewoRTUimc1stP//796/fOcopV3s0tEf2yK1lEH2PNQRFIGAQCp/BSq6A24S+c0ktZsWIF+E4QYIR1rn+cjYCShbPbR0unCLgaAb4syl/oTKc5y95Ow39Wln88PXYeAkoWzmsTLZEi4BkEuDKa5NChQwfrm5pHNKW9+Kd/Pg74z7fA+p3RzCnx0naO6z/xsNcaKwKeR4CrwnNycixNItpEQTBJFr+aCPTSncUj/mwpWUQcUk1QEVAEFAHvIaBmKO+1qdZIEUh4BLplAH+cDPTpBMzfCPxjjlnLQc3jtinAP+cC54wEhnQF5m4AHpLwyX2B04cDxeJmeXEJsHSzgfGC0cDKrUBv0Va49mFjGfCw3M9V3j8ZC3SVvN7+Hnj3B2/DrpqFt9tXa6cIJCQCdx8BZKUBs9YYUnj6FANDsrhMSBJPn8x3pQMLhEiuOMCcX3MgMEeIo58QzCPH+2Cb1AdgekcMAD6X9MbJQvTHTpQ4JwBl8mLDFUIkDx0LjOruu8eLR6pZeLFVtU6KQIIj8La8y+jXHxoQSABvnA0c0hv4Wo4p1ATu+q85HtAFOGkIMOExozW8KWFzLzNax7ItJs7uGuCyt8y2JBtEs/iXkMmtnwlpzDfhE+VV60cKmSwqMede/NvOi5XSOikCikBiI/DJal/9v5EOfHtl45E/r9lSKEs9lonJieYlSmm5+c7taL7597tNvv2rGJ/y6Wrry/qzRe7xj+8L8c6RkoV32lJroggoAvUIbPCbNkvfwjbpzFOTffBsM7uyN1woq244lJcf+Y7to60B8Xl9p5igEkmULBKptbWuikCCIDC8m6+idEz37QysMltU+QL0qEUIKFm0CC6NrAgoAm5AgDOYcjPM5zpxXBeJpjF9pRtK7twyqoPbuW2jJVMEFIFWIvDxauCzC80ivXU7gEvEOV0hTuo0P1NUK5NO2NuSZCfIIBa62OOxdOlS1NbWYsQI3VQs9uhrjk5CYNasWRgyZAi6d/f4XMx60Ac/GJ33WaSI3SQn3aybcFL7hlsWrul4XKboOkXUDOWUltByKAKKQEQRqJZ1FFxgpxIZBJQsIoOjpqIIKAKKgKcRULLwdPNq5RQBRUARiAwCShaRwVFTUQQUAUXA0wgoWXi6ebVyioAioAhEBgEli8jgqKkoAoqAIuBpBJQsPN28WjlFQBFQBCKDgJJFZHDUVBQBRUAR8DQCuoLb082rlVMEnI/AHVNlR1dZE6HSGIGCrMbn8T5Tsoh3C2j+ikACI8AdYU+Wd0k4RVauXIFOnTqja9eujigS8WknL2xygihZOKEVtAyKQIIiYHWEDukM2QTbtpYiNaU9knOdQRZOeizUZ+Gk1tCyKAKKQFwRSE5Oltetqk0sWCMoWQRDRa8pAopAQiLQrl07JYsmWl7Joglg9LIioAgkHgJKFk23uZJF09hoiCKgCCQYAkoWTTe4kkXT2GiIIqAIJBgCShZNN7jOhmoaGw1RBBSBBEGguroalZWVqKmpsXwWpaWloLO7U6dOCYJA6GoqWYTGSGMoAoqAxxGYN28eKioqGmq5aNEiJCUlYdKkSQ3XEv1AzVCJ/gRo/RUBRQA5OTkWOdhQkCg6d+5sn+q3IKBkoY+BIqAIJDwC+fn52LNHlkvXC4+7detmn+q3IKBkoY+BIqAIJDwCWVlZyMjIaISDkkUjOJQsGsOhZ4qAIpCoCBQUFDRUPTMzE6mpqQ3neqCahT4DioAioAhYCOTl5TX4LXJzcxWVAATUDBUAiJ4qAopAYiLQvn37Bj+FmqD2fgZ06uzemOgVRUARaA0CtZVA2frW3OmYe/p0aYeUukxkVG8EtjumWK0rSIq8EKNDjtiPklt3f8BdShYBgOipIqAItBKBzYuBZw5o5c3OuC1TijHIGUVpeylGXARM+1fb06lPQc1QEYNSE1IEFAFFwLsIKFl4t221ZoqAIqAIRAwBJYuIQakJKQKKgCLgXQSULLzbtlozRUARUAQihoCSRcSg1IQUAUVAEfAuAkoW3m1brZki4GoEqmqAimpfFQLPfSHOPJq3Dvjrp6ZstfJa71umA6u2tL6sD34OzF7T+vvbeqeSRVsR1PsVAUUg4ghs3Q3s+xdgzTaTdOB5xDOMQoJzA8jitg+FLEpbn9EDQhZfFrb+/rbe2b6tCej9ioAioAhEGoFt5cCyTb5UA899Ie44SpWetvrP7ihrU6VUsmgKGb2uCCgCUUVgpZhk7p8FLC0BMmTPvgP7Ar88VDrVWuA375qsf/cecP5Y4Ll5vvPLJgKT9wF++orEmwr860tgviwcHyQ7iv/2CKBHdvjFfmsJ8JqsJSzcCvSUl+Kdsx9wzDBzP80+/WUBNEnro+XAyHzgwnHAsDwTzlH+jO+Bg/uZMpAQThsFnDRy7/xrpE5XSnmvnQwM7W7CF0iZH/rC5D1c0vz1FKDA78V805cBLy4AdsjC+Esn7J1mrK+oGSrWiGt+ioAiYNnuaWYqLQd+JIu+2dH/Ucw0v3sfaC+7U4wuMCCN6iGduHT+/uf5WQB9AI/OBo59FCjeCZw4wnToLVmwzI763GeBAUIIF0gZdldJeo8BX9f7BT6QzvrCF0yHfbaQyGLZAWTKP4D19duALBcSuWcm8OPngfF9AJaL6T0/f+8GrpNXZbC89r0fC/kc+IDsjiJ5njEa+EryJB4b6tN+f6nU6XGAb9g4oBdw8YuGVPZOOXZXVLOIHdaakyKgCNQjwNH6WdJJPnKGbF0kQ9bzxgCbdhmbfJr0Suycb3xP4sj34FwgJ6Pxue34PlPSuPVok+gQiXfkv4CiHeFpFyVlwF9OBC4XTYVy7v5A7s1SBum4x0nnT6kSjeCLnxsCI6kNvAP400eiEZxqwjnqf/58IZl6bSQ5CbjqNdFQJK3m5Lq3RIMZau5lPGpLY/5q0n5Q0r76DamvaE03TTOpsJ6D72ouxeiHKVlEH2PNQRFQBAIQOFo6SpqSPv4B+K4E+LYY+PB7GZ23wITEJDmit6VPF3O0S0br4cgtRxliefUbMYUJeS3cAJTL7CubiJjGEYMNUdjpTZNzOq5tSRMtiPWwZdoQ4M5PgLXb7Ct7f1fWSF5FhtBueMcXniykOUfS3iUEtHwzcPggX1j/roY0fVdif6RkEXvMNUdFIOER+EY65qMeAVKlsz20v9j95VMmnaS/UzsckDqm+mK1k1E9xe/tqOZCE3/v/QxgZ01T18H9gNP3Bf63unHkfvUEZF+lhuNPRrmy8yD9LbYwnMK6NCU7KgCapTLTRKuqLzPjHjkY6JIO7JR7GU4/h7+kCJnEU5Qs4om+5q0IJCgCf5huHL0f/gTgiJry+Srji+BxUkDHH3jOOG0RahD/J0RxzwnALw4xKdEP8qPnTEdtp03Nx1/o0B7T03dl3XbRAkQrGSQmMAq1IxIYTWKsTzAhwWQLURSIFvWnY30x6NBOEfKkdkX/x3RJa/JAE75JTGbfiDYST4kzV8Wz6pq3IqAIxAsBdoalspaCnTY1gTcWAy+LOYgmGoo9QqfJZ7s4wQPPTazW/20vPV9X0QI27hByEJKgc/uq1yV/Gc1X1JeBqdM09cRsU05+L5BzOsP95dYZQIk42b8qBB6TOBdKOP0wzcmVBwFPzwXeXGII8rMVMovqCWDzLnMX03huPjBTyIprTG7+wDi7m0sz2mGqWUQbYU1fEVAE9kLgmsPM7KK8P8j7eaQX2l9G63cfb0b7NNNkdwCOHiKObxnpXzPJOKL9z28/Zq8kW3SBI/g/S37shB/+0jiyrzzQONY5DdcW+iNuF4f2Fa8A3ToC/zitsS+hk5STBNfnNmNSOl0c0feeZN/d9PfN04yp6vQnxScixJIn5Hn9ZDMzinexfiSO4x4zZEJfiL9G03TK0QtJ2iMSveTDT3np0qWora3FiBEjwr9JYyoCHkRg1qxZGDJkCLp3r5+Q75Y6FstQuYUvP9oiHSL9FlnS6QaTnUIc9AnYpqrA82D3tPTaOnFGU9PhlF1/OUE66t6dgb8LQXDKK81GtjmM8Z6eA/zqTenUbwVYD5YzPcU/hdDHXFPCqb+9JJ9gQmc7p9eSqFos9suP2kVGJ4hMKi2uhd6gCCgCioCYgkJ0goEkEnjeFIaBzuHAeCQfu+NvqqP2v4cL9pqTUPVo6l5qOM3l30HIhx8niJKFE1pBy6AIKAIRQ4B+jmMeaT65J84SE8/w5uN0lplJWeKIbkrYieeGILum7nXjdSULN7aallkRUASaRGBsL3E439JkcNgBT5/bfFSuvOYnUSSEzz5RYNB6KgKKgCKgCDSHgJJFc+homCKgCCgCioCFgJKFPgiKgCKgCCgCIRFQsggJkUZQBBQBRUARUAe3PgOKgCIQGQTSc4H9fhaZtCSVqqQMpO6R5csqrUOgh6wyjKAoWUQQTE1KEUhoBDr2kM2M/tomCLgwt7hkEzYUbUSNLJaYOH5sm9LTmyOHgJJF5LDUlBSBxEYgufWrx3bt2oUNGzaguLjYwpCr1wsKCmTptiyLVnEEAkoWjmgGLYQikHgIcKehTZtEixCS2L59OzIyMtC/f3/k5+cjOTlg743Eg8dxNVaycFyTaIEUAW8jUFFRgaKiIutTU1ODbt26YfTo0ejcuYkNkrwNh2tqp2ThmqbSgioC7kZgy5YtFkGUlpYiNTUVPXv2RI8ePaxjd9csMUqvZJEY7ay1VATigkB1dXWDFlFZWYkuXbpg+PDhljYRlwJppq1GQMmi1dDpjYqAItAUAvRB0BexefNmy/9APwS1iPR02Z1PxZUIKFm4stm00F5DgFNG169fL2+N24OUlBTQZFNeXm7Z8Tt1CrE/tkPAsKa9ymwmkgRnN2VnZ2Pw4MHWezmS7P3AHVJWLUbLEVCyaDlmeociEHEESAyrVq2SdywkWR+OyOvkfZ+cPup0sgg27XXo0KHIzJSXTat4BgElC880pVbEzQiwY01LSwPt+v4vr+RMoXgKCYsaA7Udf9Fpr/5oJMaxkkVitLPW0gUI5OXlYd26dZZGweK2b9/ecgjHq+hVVVVYuHChVZ4JEyZYxdBpr/Fqjfjnq2QR/zbQEigCFgIkizVr1ljHNEfxPF5Cs9iCBQvA2UzUIliuHTt2QKe9xqtF4p+vkkX820BLoAhYCHAFMz+7d++2Ouh4kQVJYdGiRZb5iURB4uI2HB06dNBprwn8rOoW5Qnc+Fp15yHAKaYU+i+ysrJiXkA61qlRcGW17TvhNwmMM5vi7UOJOSCaYQMCShYNUOiBIhB/BLiBHsUmjViWaOPGjViyZEkDSdgzs9q1M92EvclfLMukeTkHATVDOacttCQtQEAGu2IeacENLolKjWLYsGFxcWxTe+CsLDrWOfuJm/nxw3N+q1bhkocoSsVUsogSsJpsdBEQrsBby4DrZkQ3n/ikbrSL2Oc9IKpZju8JPHNKVLPQxKOIgJJFFMHVpKOLQK0wRmVtdPPQ1COHQGVN5NLSlGKPgPosYo+55qgIKAKKgOsQULJwXZNpgRUBRUARiD0CShaxx1xzVAQUAUXAdQgoWbiuybTAioAioAjEHgEli9hjrjk6FIEU+TWk+b36OfDcocUOWqxhsv/g5WOCBlkX28m046tlu6fe2U3H0RBFwB8BJQt/NPQ4YRHolAZ8cD5QUL9oOvDcbcAMzwWuGNt0qZOFLH4xXsmiaYQ0JBABnTobiIieJyQC2UIW+3TxVT3w3BfijaPqOqnvA96oi9YiNggoWcQGZ83FAQj0EZPLRftJJ5kDlFcD84qAxxfIVuCiX//fwaaA1x8EvPIdcMpQ3/lzi4FdVcDBvYFPC4Ef7Qt07wh8sRZ4bD5QxxWCYcqpku6hfQDeMmMlMH0FwPUilLE9gPNGAXmS9velwL/mAkVlJmx/2TLqwF7Al+tMnAx5vcTLUs6Zq4GfiAbBe79cD7y0BNheae7h3wmyEO6C0eb8neUAPxRqFrcfDjwyD1ix1cRZJd/58r6iIweY9SsvSL0/lzraMlxMWz+WtHqJ9rVcyvcPKV/JLjtUv72OgJqhvN7CWj8LAdrmaWbq1AF4VTrZVduAq8Rmf52QAzvrbzcZoJZuBoqlg/Y/3yQdYv/OwCX7A/ceBWzYCcwXovm13PuLceED/KuJwB8mSwe7G1hUDNwix+x8KVP7S+d/BpCVCrz7AzBOOv/pUl4SHIX50wdx51RgiZS1RjSDh48DnjoZGJELfLZGiFDSuvIAE59/WdeHjjF1KS0H/jYNuLA+P/oszhlpyIFxJwmB3SFpnzHckCDDn5bV1oOEWCkHCVG9dhbQUUiKhLOfkNf08wxpmhj61+sIqGbh9RbW+lkIDBAT01vfiwbxoRnVYxnQNR0YI51elawCt8IONt8kEo7OqW3wOs/ZIedI/LNfAZZtMaByFE4t4b7ZoUHunmEcymf8B5i9wcQv3gUctY85JnG8vhT41XRz/uwiGdVfZMjsqvfNtS6S//mvAYuFLKgNTZN706XzPk+uUaR/t7SeO7+wTq04184w2hCvUJuiU/upb0x44N/dEn7mywafJxcKIV4OHCLaFLWI3x5qtJhf1Jfl+SVCGucAPx8H3DQzMCU99yICShZebFWt014I0HxEEw5NSQNltMwR8yHS0VNrCFcqanxEwXvWi4YxOj+8u0fIdk/c7sImCt5FDYIfOtOp+fy5vpO3U/xoJXBYX/vMaBPfiuZDoWZBDefT1dap9WeLaA80j9lCcmCdbZkl2sflY40ZyTZv2WH8XlRST6RyTMvYRtGwMlKB1GSADnOanG44mDGN0Py2b559pt9eR0DJwustrPWzEBgq9vanxWRTLVoEO+w58qHd39+pHQoqdr7+ws6So/lwpJtoFhWSdzChM53CztlfNou5Klk0CFt2irbj7x9hh76zyg6VDp4X/ISmJ/+9s0gmFHb+wYSahb/YvpRMIQyapRjunz/JZ3uF/x167GUEHEMW3DvfftmKlwHXusUHgV9NMI7cc1/1dXjjCqQTrO+M7Y7W7vwDz9ta6kIxZVGDoOnL7rRp2rpmovg93jemsMn9Gmse1Cron2it9BRthXnaDm+azGpFI1m7o2UpknRIVDSb/fm/vnuZHjUclcRAwG/cEt8Kc7/82tomhl7xLZrm7gEENskovbN0nB3qh0dHDgCOHeRbhLdNOkPKqDzjZA48N6Gt/zu3SGY4ia/jT4cbpzX9H9ceaIiDI/bnxEfBGVhT+pkynTMC2L+Hb/ZSa3O+arzxb9A3c5ak+cKSxtpGuOk+I+XjTK4j+hstY7wQ7aMnGD9OuGloPHcj4BjNQsnC3Q+S00vPKaJDjpTpspeZznKx2Of/NEts8IcANLOUiTln5mrg/qPNdNLbJMz/3J4d1dp60qRzxTtmNtXMC2QqrhAE0//L/0yKd8mInc7qx6QD5mido/mbZhoHe2vz5JTYXPFhLL7CkCRnMd0u9WqN/O1LY7b753Fm9hjJ9+G5bSez1pRF74kPAkli+gmwdManIIWFhSgpKcG4cePiUwDN1VUI0Hb+xjKZ3fNBy4rduYPxW7CzDiacGloujmjbNh94Huyell5jGTgDK9BHwHToT6DWEei/aGke/vFpimJ9/P0b/uEtOW4vtgj6X1pTPmoj/zmjJblpXCchoJqFk1pDyxJ1BLZVNJ9FIIkEnjd1Nxe5NSe2s5hxmisDSaQ1HXFzeds+i+bihBtGrSfS5Qs3b40XXwQcQxZ893BVldgCVBQBlyEwqjvw5EnNF/q6GcDHq5uPo6GKgJMRcAxZdOjQwZoNVVFRAR6rKAJuQYDrE8Y84pbSajkVgdYhIBZIZ0h6uhhqRUgWKoqAIqAIKALOQsAxZNG+fXtwRpSShbMeEC2NIqAIKAJEwDFkwcJ07NgRZWUBy1gZoKIIKAKKgCIQVwQcRRZZWVnYuVM2vFFRBBQBRUARcBQCjnFwE5Xs7GwUFRVZju7/b+884OuqrnT/qfdebcu23Bsu2GBKAJuegRAcSIDkwXuhJAwJk5DJkEcIAeIMmUycZEheSGEoA4QkPAIEYwhDf3QIBvcaW7jJkiVZxWpW89vfuVzpXulK917p1nO+5Z99T9lnn73/+/iss9fae22G/5CIwEgEOB+BM5Ml8UFgZlF8lFOl9E0gppQFexZ9fX2WKYrbEhEYjgC/JRg7yTMq63BpI3G8vr7eenZLS804WsmwBDgFWN+Bw+KJ6RMxZYbiiKjU1FQ0NZmoaxIRGIFArPQ7Gc9s27Zt2Lx5s/xtI7SX+5QUhZtE/P3GVM+C+AoLC9HQ0ICJEyfGH02V2FEE+FFDRcGIOfPnz7eeXUcBUGUdRSDmlEVRURG2bNmCnp4ecDitRARijQBNpbt370Z1dTWKi4sxc+ZMPaux1kgqT8gJxNzbuKCgwNg0E3D48GHI/hvy9laGYyTA0XrsTTA0zezZs/WMjpGnLo8fAjGnLDgxLy8vD3QYSlnEz4PkhJIyMjL/5ufnY+HChZZ/zQn1Vh1FgARiTlmwUGVlZdixY4dMUYQhiTqB9vZ2qzfB3+nTp2P8eBNrWyICDiMQU6Oh3OxLSkosU1Rtba37kH5FICoEDhw4gLVr11rP45IlS6QootIKumksEIjJnkWiWRiZJqiamhpMmDAhFjipDA4jcPToUWzfvt0axj158mTwr0QEnEwgJnsWbJDy8nJr3LrCfzj58YxM3TkE9sMPP7TMnrwjV2z84IMPLCf24sWLpSgi0wy6S4wTiJllVX1xYvc/MzMTc+bM8XVax0RgzAS6u7vx/vvvW4qCw7bZq+XgCvZop06dapmfxnwTZSACNiAQsz0Lsp00aRLq6uoUttwGD1qsVmHr1q3gLGwKJ4Oyl7FgwQJMmzZNiiJWG03ligqBmFYWdHRz1bx9+/ZFBY5uam8C+/fvt5QDZ2C7hYqDvVmJCIiAN4GYVhYsakVFheXoprlAIgKhIkBfGGdheyoKTgbl7Oy9e/eG6jbKRwRsQyDmlQUd3SkpKfj4449tA10ViS4B9h4Y+I+Kwh0Kn74KTraj+YnmT4kIiIA3gZgcOutZRP4nrqystCbpsZfhXqvbM422RSAYAvSDcWgszU10ajN4JaMGuBVHMHkdg1E45o9EBOxOIKZHQ3nC51BGKop58+Z5HtZ2BAis6V2D3mMuJ3AEbheRWyT0JOBY8oCvYjQ3LU8ox0lJJ43mUl0jAnFHIOZ7Fm6iHMa4ceNGtLS0WCvquY/rN/wEao7VoNf8sZWE4MlPQ5qtkKgyIjASgZj3WbgLT1MBI9Lu3LnTfUi/IiACIiACESIQN8qCPGbMmAEGc9NQ2gg9HbqNCIiACHxCIK6UBX0WjNHDMNGdnZ1qRBEQAREQgQgRiCtlQSZcbpUT9WSOitATotuIgAiIgCEQd8qCwxu5jGVjYyMOHjyoRnQgge6j3Vizcg0O7zvswNqryiIQHQJxpyyIKTc31+ph7Nq1Cx0dHdEhp7tGjUBPZw/W/HANGvc1Rq0MurEIOI1AXCoLNhIn6nFSFQPBeYZscFoDqr4iIAIiEAkCcassaI5i6HKOjlIokEg8KuG5B01Kj1z/CD7+28f47Rd+iyduecKKz8S7vfPIO7j3intxz4p78NLdL6G3x/dcj5d/8TI+ePwDrwI+fcfT2PbKNq9j2hEBERg9gbhVFqwyR0dxTWQOpT18WPbr0T8G0buyt7sXbz3wFu6/6n6kpKWgo6nDWlPisW89hj/f/GeUzijFtFOm4YWfvoB7L7vXZ0E3/XUTdr+72+vc2sfX4sCmA17HtCMCIjB6AiGYxzr6m4fiSgYabG5utsxRXNVMsaNCQTXyeSy+dDE+d9fnrBvX7qjFa/e8hqsfvhpLr1hqHeP52+fcjh3/bwcmLpoY+QLqjiLgcAJxryzYfhwd1dbWZkUSpcJg8EFJfBGYsnRKf4H3rN1j+aH2fLAH+zfs7z+elpUGnpOy6EeiDRGIGAFbvFXpv2CAwa6uLmzbJjt1xJ6eEN4oqzCrP7f2pnYkJiciOS3ZUvxU/vx75o1nYvy88f3pvDYGxQTs7fLt3/C6RjsiIAIBE7BFz4K1TUtLw9y5c7FhwwZrhjdnekvik0Dp9FL09fRhwYVmedNTp1mV6OvtwzsPv2P5MAbXikrlaOvR/sN0hDfu17DafiDaEIEQELBFz8LNgYvX0CTF0VG1tbXuw/qNMwKzls9C2cwyrL5zNao3V6O7sxvPrHwGT373SWTkZgypTenMUmx4dgMO/f0QOlo6LMc4lYtZakIiAiIQIgK26Vm4edDhzbhR27dvR2pqqhWp1n1Ov/FBICklCTc8eQMeuu4hrFy0EqmZqZiwYAKueegaZBdno6PZeyLmud86F7ve2mU5wNnLOPXLp2LO2XOgNYnio71VyvggEDeLHwWLk76LhoYGLFq0CFlZA/bwYPNReuC+nvuitp4FFQPNStlF2X6boqW2Bek56ZZy8Zs4BAkmJ0zGp5M+HYKclIUIxD4BW5mhPHHPmjUL2dnZlg9DIUE8ycTXdkZeRkCKgrXKLcuNmKKIL4oqrQiMnYBtlQVHSB133HFWhFo6vbnmskQEREAERGB0BGyrLIgjKSkJ8+fPR3JyMtavX28NrR0dJl0lAiIgAs4mYGtlwaaloliwYAHY02APo6enx9ktrtqLgAiIwCgI2F5ZkElKSgoWLlxoBahbt24duru7R4FKl4iACIiAcwk4QlmweTmMliOjGM6cCoOzvSUiIAIiIAKBEXCMsiAOt8KgSYoKQ07vwB4SpRIBERAB203K89ekNEmxh0GHNxUG/RmKVDsyNROZyUyG1nTowZQSNOtvMBLt25iAbSfl+WszOro3btxoLcvKEVM5OTn+LtH5AAlwbZGmpiZMnTo1wCviNxmVqJRG/LafSh44AUeZoTyxcJQUnd5cz5u9DC2e5ElndNtctZAKmH8ZcsUJy91KUYzuWdFV8UfAsT0Lz6basWMHampqrCCEjC0lCY4Ae2kM3lhdXW3Nmp82bRry8vKCy0SpRUAEYpqA43wWvlqDkWrp/GbwQTq9Fd7cF6Whx9hzoIKgouB6E+QoZTuUk46IgB0IqGfh0YoHDx7Ezp07UVJSgtmzZ1sT+TxOa9ODAM12u3btssxNFRUVloLVCoUegLQpAjYjIGUxqEHpmN2yZYs1QoqxpTh6SjJAgH4JKgkqCypVmpy48JREBETA3gSkLHy0L6PUbtq0Cb29vVYwQkavdbrQL1FVVQX2vshj+vTp1uAAp3NR/UXAKQSkLIZpab4c2cNoaWkBw53zK9qJQr/EgQMHrKVqGZhxypQpKCsrcyIK1VkEHE1AysJP89Pksn//ftAuT5OLk4SLR+3evdvyS0ycOBGTJk2yHNlOYqC6ioAIuAhIWQTwJNTV1VkjpWh+mTt3rjVyii9ROnQrKysDyCF2k7DnwPqVlpb2F7Ktrc3ySzQ2NlrHOblOfol+PNoQAUcSkLIIsNnp2N28ebMV4nzChAmW/Z6XMlxIQUFBgLnEXjLWqb6+3hr9VVhYaA2DpV+CM9rZk+KkRYkIiIAISFkE8QzQ4c21vTkSqK+vz7qSo6WWLl1qrZsRRFYxkZS9I5rY2LvgjHaK/BIx0TQqhAjEHAHHhvsYTUvwRcpJe3y5uoWOcM4AjzfhZLp9+/b114X1oJmNik8O7HhrTZVXBMJPQMoiCMa07R85csTrCrfNn+fiRdgz4uTDwdLc3KyFoQZD0b4IiIBFQMoiiAehuLgYc+bMAX/Zy6BwbQwKexfxsKBSa2ur5XuxCm3+GTzrms5tiQiIgAgMJiCfhQeRrl6PHT+b7FG0mC/xxsMNONxQZ5mnysdNwJRp0/1cGd3Tmzasw5GWZkvZZWXnGNNTDjKzspCVlY2MzMx+5eerlKku/ejrlI6JgAjYnICUhUcD72wAHt3kcSCIzUy0owfJ6EJqEFdFPmmqKSH7QkeDLOcls4F5ZnRtkqsjFfmC644iIAJRJaCosx7497UAD67zOBDUZmZQqaOXeHTKbL5RFFQWEhEQAWcSkM/Cme2uWouACIhAUASkLILCpcQiIAIi4EwCUhbObHfVWgREQASCIiBlERQuJRYBERABZxKQsnBmu6vWIiACIhAUASmLoHApsQiIgAg4k4CUhTPbXbUWAREQgaAIaJ5FULhCl/j0ScCFM4DCDODp7UBBOlDTCrxUBVy9ENjdBJxh0hSb6Ru/eA9YXgnUmkgcz3qEdPr2KcC7+4G39oWuXMpJBERABHwRUM/CF5UwHzuzErj/IiDZ0P+gGrjtdOCWT5m1MT5ZrXSZOf/js137WWYOXVs3cJY5tmQcvOQzRtnMLPI6pB0REAERCAsB9SzCgnXkTH+wDHhsM/D911zpXv3Y9Ciu8r6mswe4/AmgbyAauncC7YmACIhABAmoZxFB2LxVXhowOR+ggnDLzsMAQ414ysZDUhSePLQtAiIQXQJSFhHmn2d8E5Rq45/wlOZOzz2gscN7n3uDY/ilKArsUEg6IgIiEBYCUhZhwTp8pgdMD6Ld+CCOKxlIQyf2PI/9gTMDW0dN+PTMlIF9Rn8dlz2wry0REAERCCcB+SzCSddH3r3GB/HAR8B3jEM7xajqbSYs+o0nchElH4k9DlWZ0VErZhkTVh7QYHodHAlFheHnMo8ctCkCIiACoycgZTF6dqO+8mfvupTDPy0F0kwLPL4FmJhr1pgYYfGlez8EThwPvP5lgM5vXvPmXkD+71E3gy4UAREIgoCURRCwQpX01AqAL/+fvO3Kkb0DLi5U3+7a//LTQ+90qA24+DHXvIvWLpfCGJpKR0RABEQgPASkLMLDdcRcv3ES0GH8Ft9/1fyaXsK1xwPpxh/x+p4RL7NOuhWK/5RKIQIiIAKhIyBlETqWAed0y0vATSebmdtXuJzWHCZ71VPAwUEjpALOUAlFQAREIMwEpCzCDNhX9gzl8Y3nXc7pJOPk7unzlUrHREAERCB2CGjobBTbgs5pKYooNoBuLQIiEDABKYuAUSmhCIiACDiXgJSFc9teNRcBERCBgAlIWQSMSglFQAREwLkEpCyc2/aquQiIgAgETCDhmJGAU9s8IWM21YZh+Gpfbw8SEpNMSI+xB+fo7e5CUopZ5CLCUpLlGuabOPYqRLjkup0IiEAoCEhZhILiCHl0d3dj/fr1yM/Px/Tp00dI6f/UoUOHsGPHDpxyyilISlLIWf/ElEIERCBUBGSGChVJH/l0dXVh3bp1YOdt0iSzRuoYpajItSxebW3tGHPS5SIgAiIQHAEpi+B4BZyaioI9CsrChQuRmjp20xF7E6Wlpaiurg64HEooAiIgAqEgIGURCoqD8nD3KOijWLRoUUgUhfsW48ePR1tbG1paBi2t506gXxEQAREIAwEpixBDPXr0qGV6SkxMtHoUKSkeKxaF4F7Z2dnIyclR7yIELJWFCIhA4ASkLAJn5TclFQVNTzQX0fQUakXhLgB7F3V1daDzXCICIiACkSAgZREiyu4eRbgVBYtLvwV7LjU1NSEqvbIRAREQgZEJSFmMzGfYs729A8vadXZ2Wqan5ORkq0fB33AKFUV5eTkOHjwYztsobxEQARHoJyBl0Y8i8I329na8+eab2LdvH6goaHqiyYmmp3ArCncpx40bh46ODjQ2NroP6VcEREAEwkYgvJ/AYSt2dDPes8e1pN3u3bvBOQ9uRRHJiXKZmZnWRD8Ooy0oKIguEN1dBETA9gTUswiyifk1z5nUbuEw1uLi4qjMqKaju6GhARyqKxEBERCBcBKQsgiS7t69e4fEeKqqqrJGJwWZ1ZiTU0mxV6NJemNGqQxEQAT8EJCy8API8zT9ExyB5I696A4MmJeXh6wsE2kvwsL709GtUVERBq/biYADCchnEUSj06FN4UuaCoO+gsmTJyM3NzeIXEKblKYolqu+vt4yh4U2d+UmAiIgAi4CEVUWfSYYerzGQ+cEOPdQ1SJj/pk4aXJ/b6I3RJVKGkX477S0NBQWFlqmKJqlJCIgAiIQDgIRVRaH2oC3XB/n4ahLWPNMQDLSs6agK60E9Qnp2Bbiepw4HphgOiijURjsXWzatMkaSpuRkRFWDspcBETAmQQiqiy21AH//EK8guZn/8SwFf7n57mUxWhuwJ4Fexjs+UydOnU0WegaERABERiRgBzcI+KJn5OcpOfpfI+fkqukIiAC8UBAyiIeWimAMlJZMASJ5xyQAC5TEhEQAREIiICURUCYYj8R51vQwa05F7HfViqhCMQjASmLeGy1YcpMRzcXReKscokIiIAIhJKAlMUINLPHvhLqCLmH/hQnBzJmlHoXoWerHEXA6QSkLIZ5AlYuB65eOMzJGD7M3gWDG3qGUI/h4qpoIiACcUJAymKYhjq+fJgTMX6Y4T8oVBgSERABEQgVAdsqi+JM4NbTgN+vAP7DzGE4Y9IAMk6A+6455ynnTQVuWOI68pXFQIWZIHfeNOBrJ3imiv1thkkvKSnpn20e+yVWCUVABOKBgC2VRV4a8NwXgTMrgRergEQzn+6BzwJXznc1yXSz/MOKWd7Ns6AMOMcoDMquw0C7Wd66phXYabbjTWiKam1ttZzd8VZ2lVcERCA2CUR0BnekEHztRCDLOKc/9SDQ3Qc8tN68+M0AoVs+BTy+xX8pXvkY+FYHsMFYcl7c7T99rKXIyckB/9LRHc0gh7HGReURAREYPQFb9iyOKwHe3OtSFG40L5mXfo7pcUxzyKJy7F3U1dWhp6fHjUC/IiACIjBqArZUFrlGKdCE5Cn17a69pE9qPDjAa7LNSJSWliIxMVFrXXg+BNoWAREYNQGbvSJdHD5uApZXejNZPhnoMSap7Q3A0V4gM8X7/KQ87/1436OiKCsr05yLeG9IlV8EYoSALZXF7zcClfnA9WZUU5ZRCkvN6KcvGec2/Q9dRlFUGWVCk9SlcwD2KM6ZApxt/npKYycwvRAoNaOq4lVoiuKa4Y2NjfFaBZVbBEQgRgjYUlm8dwC4+UXg68bR/dFXjYPbDJ9lePRvPu+i/lEN8OA6YNU5wI6vAzedDPz6b94t8t+7gAtmAKuv8D4eT3uczZ2fn6/eRTw1msoqAjFKIMEsDxqidd781/AVM4z16tX+04UyxbhsoM74K2iCGizpZiwYQ3q4/RmDz6cYVUofR2cEfMRcz2LF7NEtfjS43J77dHJv3boVJ598MlJTTWUlIiACIjAKArbsWXhyOGgc3b4UBdNQCQynKHiew24joSh4r3AJI9EyIq17Sdhw3Uf5ioAI2JuA7ZWFvZvPf+0SEhLAECBSFv5ZKYUIiMDwBKQshmdjmzNcGKmrqwv19fW2qZMqIgIiEFkCUhaR5R2Vu6Wnp4PrdKt3ERX8uqkI2IKAlIUtmtF/Jdi74BDazk4zJlgiAiIgAkESkLIIEli8Ji8qKkJaWpqG0cZrA6rcIhBlAlIWUW6ASN6evYuamhpEcLR0JKune4mACISRgJRFGOHGWtZUFlxBj3MvJCIgAiIQDIGIhiifa6LB/uzcYIrnnLRckCncwvkWNEcxdDkDDUpEQAREIFACEZ3B3cu54hGbL+4bQVNTk/myPoQZM2b6ThDFo+6IuOEsAuu/fv16nHDCCcjKygrnrZS3CIiAjQhEtGeRxLjg/BtFaWttQUtzkxXGI4rFiNqtGSuKMaPYu5gxwwS/koiACIhAAAQc57NgFNaMjIwA0Ng3CaPRHjp0yPJf2LeWqpkIiEAoCThOWXCeASepOVm4zgVHRFFhSERABEQgEAKOUxbqWZg1PJKTUVJSojkXgfwPURoREAGLgKOUBb+mGSPJ6T0LtjxNUa2trWhpadF/BREQARHwS8BRyoImKCoMp/ss+FTk5ORYf+noloiACIiAPwKOUBYcLnrgwAHU1tZaPLQIkOuxYO+CE/R6eiKwupO/J1HnRUAEYppAROdZRIvEli1bhsxaZpyk448/3oqXFK1yRfu+fX19eOeddzB58mRUVFREuzi6vwiIQAwTcETPgiYXLgLkKQx7wRnNTpbExERwZJRMUU5+ClR3EQiMgGOUhWfwPCqOyspK8GXpdKEpiiPEGL5cIgIiIALDEXDE25I9C0/h0FG+JCWwZnNzVrcWRtLTIAIiMBIBRyiLpKQkr+Gy7FUMNkuNBMnu5xiNlkuuclixRAREQAR8EXCEsmDF8/LyrPpzJBRfjpIBApygR/+NehcDTLQlAiLgTcAxysJtilKvwvsB4B57WeXl5VIWQ9HoiAiIwCcEwhJ1ls7kDvOn2/yJFUktTEVhWSEyyjLQfKw5qsXKRW7MmcHY29q3bx8aGhqsNS+iCkg3FwERiDkCYVEWrOXbfW9j17FdsVNhjpKdbv72Rb9I1yZdi2TzJ5aEIVAKCgqsYbRcIEkiAiIgAp4EHGOG8qy0tn0T4AgxDqFlWBSJCIiACHgSkLLwpOHwbfYoOLNdk/Qc/iCo+iLgg4CUhQ8oTj5E30VNTY0VcNHJHFR3ERABbwJSFt48HL/HUVEMLMgAgxIREAERcBOQsnCT0K9FgPNQiouLZYrS8yACIuBFwBbK4qO/fIRHvvoI/vL9v1iV62jp8KqkdoIjQEd3c3Mz2tragrtQqUVABGxLIO6VxZ61e/C7L/wOVBD54/Kx8a8b8csLf2nbBotExRgrKjMzU5P0IgFb9xCBOCEQ98qienM10nPScd0frsPyry1H9aZqdLUpxtFYnz86urlYFNe8kIiACIhATMwM2/vRXrz6q1dxeO9hFE8txmnyJjKXAAAQKUlEQVTXnoYpS6f0t86ut3fhjf98A03VTRg3dxzO/edzUTixEB899RHeevAt9PX24dEbHsXERROtY437G/HI9Y/gkh9fgvf/8D7KZpahvqoeG5/biPwJ+Tjv2+chITEBL/7sRRw5dARLLluCJZcu6b9f3e46qzw122qQmpmKqSdPxVnfOAvJqcnY9PwmrHt6HVb8cAWyi7Ota166+yVLQV3wvQv684j3DTq6q6qqLIWhWFrx3poqvwiMnUDUexYttS34+dk/R3J6Mk677jQrDMaqM1aBL2rKhjUb8NPlP0VHcwcWX7IYVBwrF60EX+gFFQUomlxkvcQrT6xE8ZRiSxnwBc99vty3vLAFD3/lYSufOWfPwa63duHXK36N3176W6TnpqNsdhnu++J9oDmLQqXyw+N/iLbDbTjpypNQOqMUz971LJ6+/Wnr/PTTpmPbK9vw2E2PWfss35O3PIlZZ86y9u3yD8O4l5aWytFtlwZVPURgjASi3rOgGamrvQsX3XERcktzccJlJ6B8dnn/OH++lE/84om45qFrrKqecf0Z+N7072H17atx7e+vxYzTZ2D7a9tx+nWnW+cPbDyAul11/fs8mJaVhhuevAFJyUkomFhg+Tg+v+rzOOemc6xrdr6+E+ufWY/JSyajdketVYYrf3dl/+JIR+qOoOq9KittenY6rn34Wqxatgqzz56Np259yir7tFOnWeft9A8d3R9++CGOHDkCdyBGO9VPdREBEQicQNSVReXSSpRMK8Hts2/H3HPnYt6n5+Hkq05GdlE22hrb0LCnARf/68VeNZp/4Xyrx+B1cISdigUVlqJgkpKpJVbKeefP67+C5qSWmpb+4zOXzcT2V7ejZmsNqrdWY9vL25BbntuffspJU3DBrRdYI7DYozj/f5/ff85OG1QQ2dnZVu9i1ix79Zzs1E6qiwhEgkDUzVD8Ur/lnVuw4q4VVg/jjzf+EbfNuM3qLXQ0uYbAFowv8GLBHgj9FIFKVlHWkKQ0QfmS/Rv2Wz0Xmq6q3q9CxfwKzDhjxpCkGfkZ1rHent4h5+x0gL2LQ4cOWRP17FQv1UUERCA4AlFXFjT70O6//IbluPGZG7Hq4CqUzynHq/e8isJJhUhKScLmFzZ71Wrzi5tRsbDC65h7Z6wr4K1ZuQbls8px1867cM3D12DZPy7Dsb5jXsqJpjOan674xRU4uOUgXlj1gvv2tvstKyuzzHEMASIRARFwLoGoKwuif+iah7D2ibXWC7mzpRPtje2WaSoxKRGnf+V0vPfoe9b8ia6OLrxx3xuoercKSz4/MHrJs/kyCzPRfLDZ8j2M5quf5iaav3gvrsuxbvU6fPjEh+g52mPdpqerBw/8zwew4DMLrKG6l999OVbfuRoc0WVHSUxMBBWGVtGzY+uqTiIQOIGo+yw4rPXSn1xqOaypNPhSXvjZhZZPgNVwm6c4gokO6uySbOuL/sTLT/RZSzq8qWTumHcHvvPGd3ymGekgnd7sOdw8/makpKdYw3Ev/fdL8eR3n7Qm/j33o+esIbzffP6bVjZLr1iKtY+vtRTIre/fitSM1JGyj8tzHDp74MABNDU1gRP2JCIgAs4jkGC+no+FutrM8uW+l4Ne/Ki5phmZ+ZnWS3pwmbqPdqO1rtUaLjv4nK/99qZ2Ky9f5wI51trQag295YS/UIu1+FFC1PV0UNVat24dGDdq7ty5QV2nxCIgAvYgEBNmKDfKvPI8n4qC51PSUgJWFExPpTMW4WiscCiKsZQpmtfS0V1fX4+uLs2Oj2Y76N4iEC0CMaUsogVB9/VPoKSkBJyoJ0e3f1ZKIQJ2JCBlYcdWDUOdOMqMIUDk6A4DXGUpAnFAQMoiDhopVopIU9TRo0fR0NAQK0VSOURABCJEQMoiQqDtcJv09HQUFBQoXpQdGlN1EIEgCUhZBAnM6cnZu2hsbERnZ6fTUaj+IuAoAlIWjmrusVe2qKjIGkIr38XYWSoHEYgnAmEb7D8xYSIyzB/JUAIJSBh6MI6OcJJedXU1KisrrZDycVR0FVUERGCUBMIyKa8PfThm/kRb6uvqkZySHHOzjqksEs2feBXOtXj33Xcxe/Zsa82LeK2Hyi0CIhA4gbD0LGLlRVi9vxq5ubkoyi8KnIhS+iXAmdzFxcVW74ILJElEQATsTyB+P28DaJuOjg5kZMgUFgCqoJPQFNXc3Iz29vagr9UFIiAC8UfAtsqit7cX3d3dUhZheiY5hJaKmL4LiQiIgP0J2FZZsFdB4dwASXgIcBhtbW0t+voCX4gqPCVRriIgAuEmYFtlwXkADFEhZRG+R4jhP6gouJKeRAREwN4EbKss2LNIS0vT0M4wPr8MLMgAgzJFhRGyshaBGCFgO2XR0tKC1tZWyLkdmSeMpqgjR45Yf+nsZhhziQiIgP0IhGWeRbQwcdGl119/vf/2XBI0MzMTnHXMCWSS0BMg8w0bNliDCdra2qwbLFu2LPQ3Uo4iIAJRJRCWeRbRqpHbR+GOW0R7OnsZnBcgCT0B+ip27tyJnp4eL3MfuVNRS0RABOxDwHb/o/Py8rxeXGyqKVOm2KfFYqgmHJ5MRUHxXJ3XfSyGiqqiiIAIjJGA7ZRFTk5OPxL2NGiCys7O7j+mjdAR4MS8ioqKIRlSiUhEQATsRcCWysL9lctf9SrC+8BOmzbNCv1BxewWKQs3Cf2KgH0I2E5ZePYiGL8oKyvLPq0VozWZO3euFYPLrTCkLGK0oVQsERgDAdspC/cIKDJRr2IMT0YQl1JJHHfccf0TIOWzCAKekopAnBCwnbIgd0ZCpT2dw2ajLsYUFnE5FnmfASfoLVy40GKu0WcRb3HdUATCTiA08yx+VQD0aJnNIa015QLg4ieGHA77gT6jLF78KrD1D2G/VVze4J9agKSUuCy6Ci0C0SIQmnkWVBS9UhZDGrGve8ihiB3oNfdWmwyDOwq9vWFKosMiEC8EbGmGihf4KqcIiIAIxAsBKYt4aSmVUwREQASiSEDKIorwdWsREAERiBcCUhbx0lIqpwiIgAhEkYBtlEVLiP3rXSbkUWcU/dPReiY2mFVSV706/N03HgR+MsL52iPAD14AWo/6zsPfed9X6agIiEC0CdhCWdz4JPDLN0KHsrEdWPAzYG9T6PKMl5zWG2Xx7yMoAyqLn742fG1qjLK4cwRl4e/88DnrjAiIQDQJ2EJZvLc3tAibzPLd2+tCm6ddcvvSYuDQD+xSG9VDBEQgUAKhmWcR6N2GSUfTBL9W+VVbZoLGXrkEOH+WK3GHMQV94ynglrOAacWuY/vNFz9NHasuAu5/D/j4MPD0ZjPPyqi+754N/OpNE+qj0PXCf3kncFw58OUTgTllruvvfh0Ynwtctsi1z3+//zywfBpw8iSTx3Ou47f9FfjKycC5MwfSOWVr7X5Xb+2w6WWdPQP45ukwod+B9/YAf/wIuHuFiwTNdfe+a+YA7jBM84DPzPEm5O88U687ANzzFrCnEZhr2ug7Z7ry4jm25YwS4EAzsNq0cbp5Yq87CTjHgW1CHhIRiBYB83qNrtDks/g/gOe2AZ+dB/SZ+VIX3Q/85m1Xufiyue99oLZ1oJwN5hoea+8CZpcCWWnABPOi4ouG8t/bjXL4E/DYOuAKoxA21QBn/sb1wuH557YC75iXnqf8X5OW6ZKTgIXjXWfmjwPKjfJymjR3Al/8PcD6U8He+tyAaervDcAjaweIfN2YAP/1JeDUSoBt9T/+MHCOW/7Ov2KU+Sn/x/g4TFt+YaFRRntdJsBqoxwobMvr/wz8199cSqvXPB/n/yew2bSVRAREIHIEzHdadOXfXgGOHAWqbgVSTWluPM314r/lWVdvwF/pLpzrspGfUGEiaxw3kLrLRLx460bXy/+qE4Dp/wb86GXzBXvJQBpfW2mmDFQwt/4VuNz8zjRftU6Tnj7gT1caJW6YUvjF/9rfXb071xHXv3SG32+U9qZ/MYq63HVsnvn99jOBnWeqfzFp/2G26a2Y+1HYk1v8c1db/eqTtspKNfe/AWb1PaN8TjWxv+4E2GPkvSQiIAKRIRD1nsWH+41JwZg5qCjccpFRAC1GgWw/5D4S/C/NFOwluOU8s0/TisQ/AZp6Fn3Su2LqxRNMr6xl6HXrjLKg2dCtKJji05+YD7nt7/zRHmN6PAjwlx8H7r80J37g0Vb8EHCv0spf9iLZE5GIgAhEjoDHKzpyN/W8E00ebl+C+zhfQBSaHNziGby12/Qa/EmliW3oKYWZQJvHC8YzP6ZjT0TiIsAveffLmUcSja/Cl3AgAM2GZOle+8hTQfs7z+HOvD7bmBE970EfUUHGwB1ZHk9JGqY8nmm0LQIiEFoCUVcW043T+vlt3pXifrL5gqRjmi8TiueXJB3a/uSVv3unoAOWX8gUmpo88+sximL/JzZynne/+AYrFJ6TDBCgmeqQ8SWxB3H8J2xpHnKLv/MlZrXbXKMoONjgRxe4rwJeMH6KFI9e4cAZbYmACESLQNTNUP94CkCnKSeCHTFfmq/vAn73rsvZzZd6RgpQYcwOHPXE87tNWjpUPaXI9Bq2GpPVQQ9TCUdWPfg+wNFU/OUL7X8Z3wWFfog1W8x96425y+RJG3uvsdO7lQN7IRSarZrN17PENwGah+YZB/jKF4yybQLow+DIKLf4O890N5zqcphzpBPbgO1/8YNAfZs7F/2KgAjEAoGoK4tlZrjq/ZcBdHSX3AFccL/LXv7olwbw/OZSl0Oz4PvACXcbp+jygXPcWmEc23/eACz9xcBxDoO962Ug/zbgtufN6CqTx1nGN0L59jJgsjFTzfixsbnfCdCsRb+Ju0eRm+6yvXNkz8oXrUv0jw8C6UaRr7kWqDZKmgMIyP+C2QMJ/Z1nyjvOcw0o+PxDQM6tRqH/Cbh5uWtk1EBO2hIBEYg2gdAsfnS3MTCPce0EftXTFMShqr5MEH3mq5NO1gnGZOFpT3cD5LBNjuLJTHUNvZ2YD/zaKAiOz6eZw60I3On5y/kdOcYMwmt8CXsyPEeH66hk6kXA51aP6tIxXcTFj56/2nS3HhlTNsFcXGfMUWRJBeFL/J2nwmZ7VJh2C7vcZEZPJA3T6GG/uW4gAvFJIOo+Czc2vsz5gh9OqCBGOs/RVL7++3PkzHDidqQPdz7H9DAkgRGg/2Ek8XeeHwgRURQjFVLnREAEhiUw2m/mYTOMhRP5pqPDr1yJCIiACIhAaAjETM8iNNVx5fKIh78jlPkqLxEQARFwKgFb9iyc2piqtwiIgAiEi4CURbjIKl8REAERsBEBKQsbNaaqIgIiIALhIiBlES6yylcEREAEbEQgNA7u8WYadq8Zuy7xJlA0x3s/knsFZpbheDM9WjKUgK9JN0NT6YgIiIAHgdBMyvPIUJuDCBwzMwUTItyB46S8RDNxQSICIiACISIgZREikMpGBERABOxMIMKfvHZGqbqJgAiIgH0JSFnYt21VMxEQAREIGQEpi5ChVEYiIAIiYF8CUhb2bVvVTAREQARCRuD/A71UmWMWrXWXAAAAAElFTkSuQmCC"
    }
   },
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Calculating the attention weights is done with another feed-forward layer attn, using the decoder’s input and hidden state as inputs. Because there are sentences of all sizes in the training data, to actually create and train this layer we have to choose a maximum sentence length (input length, for encoder outputs) that it can apply to. Sentences of the maximum length will use all the attention weights, while shorter sentences will only use the first few.\n",
    "![attention-decoder-network.png](attachment:attention-decoder-network.png)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 31,
   "metadata": {},
   "outputs": [],
   "source": [
    "class AttnDecoderRNN(nn.Module):\n",
    "    def __init__(self, hidden_size, output_size, dropout_p=0.1, max_length=MAX_LENGTH):\n",
    "        super(AttnDecoderRNN, self).__init__()\n",
    "        self.hidden_size = hidden_size\n",
    "        self.output_size = output_size\n",
    "        self.dropout_p = dropout_p\n",
    "        self.max_length = max_length\n",
    "\n",
    "        self.embedding = nn.Embedding(self.output_size, self.hidden_size)\n",
    "        self.attn = nn.Linear(self.hidden_size * 2, self.max_length)\n",
    "        self.attn_combine = nn.Linear(self.hidden_size * 2, self.hidden_size)\n",
    "        self.dropout = nn.Dropout(self.dropout_p)\n",
    "        self.gru = nn.GRU(self.hidden_size, self.hidden_size)\n",
    "        self.out = nn.Linear(self.hidden_size, self.output_size)\n",
    "\n",
    "    def forward(self, input, hidden, encoder_outputs):\n",
    "        embedded = self.embedding(input).view(1, 1, -1)\n",
    "        embedded = self.dropout(embedded)\n",
    "\n",
    "        attn_weights = F.softmax(\n",
    "            self.attn(torch.cat((embedded[0], hidden[0]), 1)), dim=1)\n",
    "        attn_applied = torch.bmm(attn_weights.unsqueeze(0),\n",
    "                                 encoder_outputs.unsqueeze(0))\n",
    "\n",
    "        output = torch.cat((embedded[0], attn_applied[0]), 1)\n",
    "        output = self.attn_combine(output).unsqueeze(0)\n",
    "\n",
    "        output = F.relu(output)\n",
    "        output, hidden = self.gru(output, hidden)\n",
    "\n",
    "        output = F.log_softmax(self.out(output[0]), dim=1)\n",
    "        return output, hidden, attn_weights\n",
    "\n",
    "    def initHidden(self):\n",
    "        return torch.zeros(1, 1, self.hidden_size, device=device)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Training\n",
    "## Preparing Training Data\n",
    "To train, for each pair we will need an input tensor (indexes of the words in the input sentence) and target tensor (indexes of the words in the target sentence). While creating these vectors we will append the EOS token to both sequences."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 32,
   "metadata": {},
   "outputs": [],
   "source": [
    "def indexesFromSentence(lang, sentence):\n",
    "    return [lang.word2index[word] for word in sentence.split(' ')]\n",
    "\n",
    "\n",
    "def tensorFromSentence(lang, sentence):\n",
    "    indexes = indexesFromSentence(lang, sentence)\n",
    "    indexes.append(EOS_token)\n",
    "    return torch.tensor(indexes, dtype=torch.long, device=device).view(-1, 1)\n",
    "\n",
    "\n",
    "def tensorsFromPair(pair):\n",
    "    input_tensor = tensorFromSentence(input_lang, pair[0])\n",
    "    target_tensor = tensorFromSentence(output_lang, pair[1])\n",
    "    return (input_tensor, target_tensor)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Training the Model"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 33,
   "metadata": {},
   "outputs": [],
   "source": [
    "teacher_forcing_ratio = 0.5\n",
    "\n",
    "\n",
    "def train(input_tensor, target_tensor, encoder, decoder, encoder_optimizer, decoder_optimizer, criterion, max_length=MAX_LENGTH):\n",
    "    encoder_hidden = encoder.initHidden()\n",
    "\n",
    "    encoder_optimizer.zero_grad()\n",
    "    decoder_optimizer.zero_grad()\n",
    "\n",
    "    input_length = input_tensor.size(0)\n",
    "    target_length = target_tensor.size(0)\n",
    "\n",
    "    encoder_outputs = torch.zeros(max_length, encoder.hidden_size, device=device)\n",
    "\n",
    "    loss = 0\n",
    "\n",
    "    for ei in range(input_length):\n",
    "        encoder_output, encoder_hidden = encoder(\n",
    "            input_tensor[ei], encoder_hidden)\n",
    "        encoder_outputs[ei] = encoder_output[0, 0]\n",
    "\n",
    "    decoder_input = torch.tensor([[SOS_token]], device=device)\n",
    "\n",
    "    decoder_hidden = encoder_hidden\n",
    "\n",
    "    use_teacher_forcing = True if random.random() < teacher_forcing_ratio else False\n",
    "\n",
    "    if use_teacher_forcing:\n",
    "        # Teacher forcing: Feed the target as the next input\n",
    "        for di in range(target_length):\n",
    "            decoder_output, decoder_hidden, decoder_attention = decoder(\n",
    "                decoder_input, decoder_hidden, encoder_outputs)\n",
    "            loss += criterion(decoder_output, target_tensor[di])\n",
    "            decoder_input = target_tensor[di]  # Teacher forcing\n",
    "\n",
    "    else:\n",
    "        # Without teacher forcing: use its own predictions as the next input\n",
    "        for di in range(target_length):\n",
    "            decoder_output, decoder_hidden, decoder_attention = decoder(\n",
    "                decoder_input, decoder_hidden, encoder_outputs)\n",
    "            topv, topi = decoder_output.topk(1)\n",
    "            decoder_input = topi.squeeze().detach()  # detach from history as input\n",
    "\n",
    "            loss += criterion(decoder_output, target_tensor[di])\n",
    "            if decoder_input.item() == EOS_token:\n",
    "                break\n",
    "\n",
    "    loss.backward()\n",
    "\n",
    "    encoder_optimizer.step()\n",
    "    decoder_optimizer.step()\n",
    "\n",
    "    return loss.item() / target_length"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 34,
   "metadata": {},
   "outputs": [],
   "source": [
    "# This is a helper function to print time elapsed and estimated time remaining given the current time and progress %.\n",
    "\n",
    "import time\n",
    "import math\n",
    "\n",
    "\n",
    "def asMinutes(s):\n",
    "    m = math.floor(s / 60)\n",
    "    s -= m * 60\n",
    "    return '%dm %ds' % (m, s)\n",
    "\n",
    "\n",
    "def timeSince(since, percent):\n",
    "    now = time.time()\n",
    "    s = now - since\n",
    "    es = s / (percent)\n",
    "    rs = es - s\n",
    "    return '%s (- %s)' % (asMinutes(s), asMinutes(rs))"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "The whole training process looks like this:\n",
    "\n",
    "* Start a timer\n",
    "* Initialize optimizers and criterion\n",
    "* Create set of training pairs\n",
    "* Start empty losses array for plotting"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 35,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Then we call train many times and occasionally print the progress (% of examples, time so far, estimated time) and average loss.\n",
    "\n",
    "def trainIters(encoder, decoder, n_iters, print_every=1000, plot_every=100, learning_rate=0.01):\n",
    "    start = time.time()\n",
    "    plot_losses = []\n",
    "    print_loss_total = 0  # Reset every print_every\n",
    "    plot_loss_total = 0  # Reset every plot_every\n",
    "\n",
    "    encoder_optimizer = optim.SGD(encoder.parameters(), lr=learning_rate)\n",
    "    decoder_optimizer = optim.SGD(decoder.parameters(), lr=learning_rate)\n",
    "    training_pairs = [tensorsFromPair(random.choice(pairs))\n",
    "                      for i in range(n_iters)]\n",
    "    criterion = nn.NLLLoss()\n",
    "\n",
    "    for iter in range(1, n_iters + 1):\n",
    "        training_pair = training_pairs[iter - 1]\n",
    "        input_tensor = training_pair[0]\n",
    "        target_tensor = training_pair[1]\n",
    "\n",
    "        loss = train(input_tensor, target_tensor, encoder,\n",
    "                     decoder, encoder_optimizer, decoder_optimizer, criterion)\n",
    "        print_loss_total += loss\n",
    "        plot_loss_total += loss\n",
    "\n",
    "        if iter % print_every == 0:\n",
    "            print_loss_avg = print_loss_total / print_every\n",
    "            print_loss_total = 0\n",
    "            print('%s (%d %d%%) %.4f' % (timeSince(start, iter / n_iters),\n",
    "                                         iter, iter / n_iters * 100, print_loss_avg))\n",
    "\n",
    "        if iter % plot_every == 0:\n",
    "            plot_loss_avg = plot_loss_total / plot_every\n",
    "            plot_losses.append(plot_loss_avg)\n",
    "            plot_loss_total = 0\n",
    "\n",
    "    showPlot(plot_losses)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Plotting results\n",
    "Plotting is done with matplotlib, using the array of loss values plot_losses saved while training."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 36,
   "metadata": {},
   "outputs": [],
   "source": [
    "import matplotlib.pyplot as plt\n",
    "plt.switch_backend('agg')\n",
    "import matplotlib.ticker as ticker\n",
    "import numpy as np\n",
    "\n",
    "\n",
    "def showPlot(points):\n",
    "    plt.figure()\n",
    "    fig, ax = plt.subplots()\n",
    "    # this locator puts ticks at regular intervals\n",
    "    loc = ticker.MultipleLocator(base=0.2)\n",
    "    ax.yaxis.set_major_locator(loc)\n",
    "    plt.plot(points)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Evaluation\n",
    "Evaluation is mostly the same as training, but there are no targets so we simply feed the decoder’s predictions back to itself for each step. Every time it predicts a word we add it to the output string, and if it predicts the EOS token we stop there. We also store the decoder’s attention outputs for display later."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 37,
   "metadata": {},
   "outputs": [],
   "source": [
    "def evaluate(encoder, decoder, sentence, max_length=MAX_LENGTH):\n",
    "    with torch.no_grad():\n",
    "        input_tensor = tensorFromSentence(input_lang, sentence)\n",
    "        input_length = input_tensor.size()[0]\n",
    "        encoder_hidden = encoder.initHidden()\n",
    "\n",
    "        encoder_outputs = torch.zeros(max_length, encoder.hidden_size, device=device)\n",
    "\n",
    "        for ei in range(input_length):\n",
    "            encoder_output, encoder_hidden = encoder(input_tensor[ei],\n",
    "                                                     encoder_hidden)\n",
    "            encoder_outputs[ei] += encoder_output[0, 0]\n",
    "\n",
    "        decoder_input = torch.tensor([[SOS_token]], device=device)  # SOS\n",
    "\n",
    "        decoder_hidden = encoder_hidden\n",
    "\n",
    "        decoded_words = []\n",
    "        decoder_attentions = torch.zeros(max_length, max_length)\n",
    "\n",
    "        for di in range(max_length):\n",
    "            decoder_output, decoder_hidden, decoder_attention = decoder(\n",
    "                decoder_input, decoder_hidden, encoder_outputs)\n",
    "            decoder_attentions[di] = decoder_attention.data\n",
    "            topv, topi = decoder_output.data.topk(1)\n",
    "            if topi.item() == EOS_token:\n",
    "                decoded_words.append('<EOS>')\n",
    "                break\n",
    "            else:\n",
    "                decoded_words.append(output_lang.index2word[topi.item()])\n",
    "\n",
    "            decoder_input = topi.squeeze().detach()\n",
    "\n",
    "        return decoded_words, decoder_attentions[:di + 1]"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 38,
   "metadata": {},
   "outputs": [],
   "source": [
    "def evaluateRandomly(encoder, decoder, n=10):\n",
    "    for i in range(n):\n",
    "        pair = random.choice(pairs)\n",
    "        print('>', pair[0])\n",
    "        print('=', pair[1])\n",
    "        output_words, attentions = evaluate(encoder, decoder, pair[0])\n",
    "        output_sentence = ' '.join(output_words)\n",
    "        print('<', output_sentence)\n",
    "        print('')"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Training and Evaluating\n",
    "With all these helper functions in place (it looks like extra work, but it makes it easier to run multiple experiments) we can actually initialize a network and start training.\n",
    "\n",
    "Remember that the input sentences were heavily filtered. For this small dataset we can use relatively small networks of 256 hidden nodes and a single GRU layer. After about 60 minutes on a Labtop CPU we’ll get some reasonable results."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 39,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "4m 23s (- 61m 28s) (5000 6%) 2.8685\n",
      "8m 24s (- 54m 36s) (10000 13%) 2.2917\n",
      "12m 23s (- 49m 33s) (15000 20%) 1.9656\n",
      "16m 34s (- 45m 34s) (20000 26%) 1.7397\n",
      "20m 42s (- 41m 24s) (25000 33%) 1.5276\n",
      "24m 36s (- 36m 54s) (30000 40%) 1.3673\n",
      "28m 25s (- 32m 28s) (35000 46%) 1.2381\n",
      "32m 24s (- 28m 21s) (40000 53%) 1.1096\n",
      "36m 14s (- 24m 9s) (45000 60%) 0.9977\n",
      "40m 4s (- 20m 2s) (50000 66%) 0.9213\n",
      "43m 57s (- 15m 59s) (55000 73%) 0.8384\n",
      "47m 45s (- 11m 56s) (60000 80%) 0.7482\n",
      "51m 33s (- 7m 55s) (65000 86%) 0.6975\n",
      "55m 38s (- 3m 58s) (70000 93%) 0.6332\n",
      "59m 30s (- 0m 0s) (75000 100%) 0.5593\n"
     ]
    }
   ],
   "source": [
    "hidden_size = 256\n",
    "encoder1 = EncoderRNN(input_lang.n_words, hidden_size).to(device)\n",
    "attn_decoder1 = AttnDecoderRNN(hidden_size, output_lang.n_words, dropout_p=0.1).to(device)\n",
    "\n",
    "trainIters(encoder1, attn_decoder1, 75000, print_every=5000)\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 40,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "> nous sommes trop occupes .\n",
      "= we re too busy .\n",
      "< we re too busy . <EOS>\n",
      "\n",
      "> vous etes fort avisees .\n",
      "= you re very wise .\n",
      "< you re very wise . <EOS>\n",
      "\n",
      "> tu es un grand malade .\n",
      "= you are seriously ill .\n",
      "< you are seriously ill . <EOS>\n",
      "\n",
      "> il doit avoir la quarantaine .\n",
      "= he is about forty .\n",
      "< he is about . . <EOS>\n",
      "\n",
      "> je ne suis pas blessee .\n",
      "= i m not hurt .\n",
      "< i m not hurt . <EOS>\n",
      "\n",
      "> tu es probablement trop jeune pour le comprendre .\n",
      "= you re probably too young to understand this .\n",
      "< you re probably too young to understand this . <EOS>\n",
      "\n",
      "> il est un peu emeche .\n",
      "= he s a bit tipsy .\n",
      "< he s a bit tipsy . <EOS>\n",
      "\n",
      "> je suis votre medecin .\n",
      "= i m your doctor .\n",
      "< i m your mercy . <EOS>\n",
      "\n",
      "> il ne m a jamais frappe auparavant .\n",
      "= he s never hit me before .\n",
      "< he s never hit me before . <EOS>\n",
      "\n",
      "> je vois mary cette apres midi .\n",
      "= i am seeing mary this afternoon .\n",
      "< i am seeing mary this afternoon . <EOS>\n",
      "\n"
     ]
    }
   ],
   "source": [
    "evaluateRandomly(encoder1, attn_decoder1)\n"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Visualizing Attention\n",
    "A useful property of the attention mechanism is its highly interpretable outputs. Because it is used to weight specific encoder outputs of the input sequence, we can imagine looking where the network is focused most at each time step.\n",
    "\n",
    "You could simply run plt.matshow(attentions) to see attention output displayed as a matrix, with the columns being input steps and rows being output steps:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 41,
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "<matplotlib.image.AxesImage at 0x1d2e147a388>"
      ]
     },
     "execution_count": 41,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "output_words, attentions = evaluate(\n",
    "    encoder1, attn_decoder1, \"je suis trop froid .\")\n",
    "plt.matshow(attentions.numpy())"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 42,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "input = elle a cinq ans de moins que moi .\n",
      "output = she s three years younger than me . <EOS>\n"
     ]
    },
    {
     "name": "stderr",
     "output_type": "stream",
     "text": [
      "C:\\ProgramData\\Anaconda3\\lib\\site-packages\\ipykernel_launcher.py:19: UserWarning: Matplotlib is currently using agg, which is a non-GUI backend, so cannot show the figure.\n"
     ]
    },
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "input = elle est trop petit .\n",
      "output = she s not late . <EOS>\n",
      "input = je ne crains pas de mourir .\n",
      "output = i m not scared of dying . <EOS>\n",
      "input = c est un jeune directeur plein de talent .\n",
      "output = he s a very talented . <EOS>\n"
     ]
    }
   ],
   "source": [
    "# For a better viewing experience we will do the extra work of adding axes and labels:\n",
    "\n",
    "def showAttention(input_sentence, output_words, attentions):\n",
    "    # Set up figure with colorbar\n",
    "    fig = plt.figure()\n",
    "    ax = fig.add_subplot(111)\n",
    "    cax = ax.matshow(attentions.numpy(), cmap='bone')\n",
    "    fig.colorbar(cax)\n",
    "\n",
    "    # Set up axes\n",
    "    ax.set_xticklabels([''] + input_sentence.split(' ') +\n",
    "                       ['<EOS>'], rotation=90)\n",
    "    ax.set_yticklabels([''] + output_words)\n",
    "\n",
    "    # Show label at every tick\n",
    "    ax.xaxis.set_major_locator(ticker.MultipleLocator(1))\n",
    "    ax.yaxis.set_major_locator(ticker.MultipleLocator(1))\n",
    "\n",
    "    plt.show()\n",
    "\n",
    "\n",
    "def evaluateAndShowAttention(input_sentence):\n",
    "    output_words, attentions = evaluate(\n",
    "        encoder1, attn_decoder1, input_sentence)\n",
    "    print('input =', input_sentence)\n",
    "    print('output =', ' '.join(output_words))\n",
    "    showAttention(input_sentence, output_words, attentions)\n",
    "\n",
    "\n",
    "evaluateAndShowAttention(\"elle a cinq ans de moins que moi .\")\n",
    "\n",
    "evaluateAndShowAttention(\"elle est trop petit .\")\n",
    "\n",
    "evaluateAndShowAttention(\"je ne crains pas de mourir .\")\n",
    "\n",
    "evaluateAndShowAttention(\"c est un jeune directeur plein de talent .\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Further Work to DO\n",
    "* Try with a different dataset\n",
    "    * Another language pair\n",
    "    * Human → Machine (e.g. IOT commands)\n",
    "    * Chat → Response\n",
    "    * Question → Answer\n",
    "* Replace the embeddings with pre-trained word embeddings such as word2vec or GloVe\n",
    "* Try with more layers, more hidden units, and more sentences. Compare the training time and results.\n",
    "* If you use a translation file where pairs have two of the same phrase (I am test \\t I am test), you can use this as an autoencoder. Try this:\n",
    "    * Train as an autoencoder\n",
    "    * Save only the Encoder network\n",
    "    * Train a new Decoder for translation from there"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": []
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.7.4"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 2
}
