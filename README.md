### Implementation of Mogrifier LSTM Cell in PyTorch
This follows the implementation of a Mogrifier LSTM proposed [here](https://arxiv.org/pdf/1909.01792.pdf)

The Mogrifier LSTM is an LSTM where two inputs x and h_prev modulate one another in an alternating fashion before the LSTM computation.

![Capture](https://user-images.githubusercontent.com/30661597/71353181-437f2080-25b3-11ea-97e6-fd52c796ad64.PNG)

```python
import torch
import torch.nn as nn
import math

class MogrifierLSTMCell(nn.Module):

    def __init__(self, input_size, hidden_size, mogrify_steps):
        super(MogrifierLSTMCell, self).__init__()
        self.hidden_size = hidden_size
        self.input_size = input_size
        self.mogrify_steps = mogrify_steps
        self.x2h = nn.Linear(input_size, 4 * hidden_size)
        self.h2h = nn.Linear(hidden_size, 4 * hidden_size)
        
        self.mogrifier_list = nn.ModuleList([nn.Linear(hidden_size, input_size)])  # start with q
        for i in range(1, mogrify_steps):
            if i % 2 == 0:
                self.mogrifier_list.extend([nn.Linear(hidden_size, input_size)])  # q
            else:
                self.mogrifier_list.extend([nn.Linear(input_size, hidden_size)])  # r
        
        self.tanh = nn.Tanh()
        self.init_parameters()
        
    def init_parameters(self):
        std = 1.0 / math.sqrt(self.hidden_size)
        for p in self.parameters():
            p.data.uniform_(-std, std)
            
    def mogrify(self, x, h):
        for i in range(self.mogrify_steps):
            if (i+1) % 2 == 0: 
                h = (2*torch.sigmoid(self.mogrifier_list[i](x))) * h
            else:
                x = (2*torch.sigmoid(self.mogrifier_list[i](h))) * x
        return x, h

    def forward(self, x, states):
        """
        inp shape: (batch_size, input_size)
        each of states shape: (batch_size, hidden_size)
        """
        ht, ct = states
        x, ht = self.mogrify(x,ht)   # Note: This should be called every timestep
        gates = self.x2h(x) + self.h2h(ht)    # (batch_size, 4 * hidden_size)
        in_gate, forget_gate, new_memory, out_gate = gates.chunk(4, 1)
        in_gate = torch.sigmoid(in_gate)
        forget_gate = torch.sigmoid(forget_gate)
        out_gate = torch.sigmoid(out_gate)
        new_memory = self.tanh(new_memory)
        c_new = (forget_gate * ct) + (in_gate * new_memory)
        h_new = out_gate * self.tanh(c_new)

        return h_new, c_new
```

The example below shows how you can use a mogrifier LSTM:

```python
batch_size = 4
hidden_size = 512
input_size = 256
mogrify_steps = 5
vocab_size = 30
max_len = 10
mog_lstm = MogrifierLSTMCell(input_size, hidden_size,mogrify_steps)

# seq of shape (batch_size, max_words)
seq = torch.LongTensor([[ 8, 29, 18,  1, 17,  3, 26,  6, 26,  5],
                        [ 8, 28, 15, 12, 13,  2, 26, 16, 20,  0],
                        [15,  4, 27, 14, 29, 28, 14,  1,  0,  0],
                        [20, 22, 29, 22, 23, 29,  0,  0,  0,  0]])

emb = nn.Embedding(vocab_size, input_size)
embeeded = emb(seq)

h,c = [torch.zeros(batch_size,hidden_size), torch.zeros(batch_size,hidden_size)]
hidden_states = []
for step in range(max_len):
    x = embeeded[:, step]
    h,c = mog_lstm(x, (h, c))
    hidden_states.append(h.unsqueeze(1))
    
hidden_states = torch.cat(hidden_states, dim = 1)
print(hidden_states.shape)
```
This outputs:
```
torch.Size([4, 10, 512])
```
