## Cell
Cell is a cell, a container for data that can store up to 1023 bits and have up to 4 references to other Cells. In TON, everything consists of cells, contract code, stored data, blocks. This approach achieves versatility.

## Bag of Cells
Bag of Cells - format for serializing cells into an array of bytes, described as [TL-B schema](https://github.com/ton-blockchain/ton/blob/24dc184a2ea67f9c47042b4104bbb4d82289fac1/crypto/tl/boc.tlb#L25).

#### Cell serialization
Let's analyze this kind of cell:
```
1[8_] -> {
  24[0AAAAA],
  7[FE] -> {
    24[0AAAAA]
  }
}
```
Here we have a 1-bit root cell that has 2 links: the first to a 24-bit cell and the second to a 7-bit cell, which has 1 link to a 24-bit cell.

We need to turn the cells into a flat set of bytes, for this we first need to leave only unique cells - we have 3 out of 4 of them. We have:
```
1[8_]
24[0AAAAA]
7[FE]
```
Now let's arrange them in such an order that the parent cells do not point backwards. The cell pointed to by the rest must be in the list after the ones that point to it. We get:
```
1[8_]      -> index 0 (root cell)
7[FE]      -> index 1
24[0AAAAA] -> index 2
```

Let's count descriptors for each of them. These are 2 bytes that store flags, information about the length of the data and the number of links. The flags will be omitted in the current parse, they are almost always 0. The first byte contains 5 flag bits and 3 link count bits. The second byte is the length of the full 4-bit groups (but at least 1 if not empty). We get:
```
1[8_]      -> 0201 -> 2 links, length 1 
7[FE]      -> 0101 -> 1 link, length 1
24[0AAAAA] -> 0006 -> 0 links, length 6
```
For data with incomplete 4-bit groups, 1 bit is added to the end. It means the end bit of the group and is used to determine the true size of incomplete groups. Let's add it:
```
1[8_]      -> C0     -> 0b10000000->0b11000000
7[FE]      -> FF     -> 0b11111110->0b11111111
24[0AAAAA] -> 0AAAAA -> do not change (full groups)
```

Now let's add link indexes:
```
0 1[8_]      -> 0201 -> refers to 2 cells with such indexes
1 7[FE]      -> 02 -> refers to cells with index 2
2 24[0AAAAA] -> no links
```

And put it all together:
```
0201 C0     0201  
0101 AA     02
0006 0AAAAA 
```

And glue it into an array of bytes:
`0201c002010101ff0200060aaaaa`, size 14 bytes.

[Serialization example](https://github.com/xssnick/tonutils-go/blob/3d9ee052689376061bf7e4a22037ff131183afad/tvm/cell/serialize.go#L205)

#### Упаковка в BoC
Упакуем ячейку из прошлой главы. Мы уже сериализовали ее в плоский массив байт длиной 14.

Строим заголовок согласно [схеме](https://github.com/ton-blockchain/ton/blob/24dc184a2ea67f9c47042b4104bbb4d82289fac1/crypto/tl/boc.tlb#L25).

```
b5ee9c72                      -> id tl-b структуры BoC
01                            -> флаги и размер size:(## 3), в нашем случае флаги все 0, 
                                 и количество байтов нужное для хранения количества ячеек - 1.
                                 получаем - 0b0_0_0_00_001
01                            -> кол-во байтов для хранения размера сериализованых ячеек
03                            -> количество ячеек, 1 байт (определяется 3 битами size:(## 3), равный 3м.
01                            -> количество корневых ячеек - 1
00                            -> absent, всегда 0 (в текущих имплементациях)
0e                            -> размер сериализованых ячеек, 1 байт (размер определен выше), равный 14
00                            -> индекс корневой ячейки, размером 1 (определяется 3 битами size:(## 3) из заголовка), 
                                 всегда 0
0201c002010101ff0200060aaaaa  -> сериализованые ячейки
```

Всё, что выше, склеим в массив байтов и получим:
`b5ee9c7201010301000e000201c002010101ff0200060aaaaa` Это наш финальный BoC!

Примеры реализации BoC: [Сериализация](https://github.com/xssnick/tonutils-go/blob/master/tvm/cell/serialize.go), [Десериализация](https://github.com/xssnick/tonutils-go/blob/master/tvm/cell/parse.go)

## Специальные ячейки

Ячейки делятся на два вида: обычные (Ordinary) и специальные. Большинство ячеек, с которыми работает поользователь, являются обычными и просто несут в себе информацию. Но для внутреннего функционала сети часто используются специальные ячейки, работа с которыми происходит по немного иному сценарию, зависящему от подтипа специальной ячейки. 

Тип специальной ячейки определяется первыми 8 битами ее данных и может быть одним из приведенных ниже:
```
0x01: PrunnedBranch
0x02: Library
0x03: MerkleProof
0x04: MerkleUpdate
```
В разделах далее мы разберем специальные типы подробнее.

### Merkle Proof
Ячейка этого типа является деревом хэшей и несет в себе доказательство принадлежности части данных дерева ячеек к полному дереву, при этом проверяющий может не знать полное содержимое дерева, а знать только его хеш. 

Внутри себя ячейка несет корневой хеш объекта, например, блока и дерево хешей его веток в виде дочерней ячейки. Само внутреннее дерево повторяет по структуре оригинальный объект, пруфом которого оно является, но при этом, все ветки не являющиеся частью данных, которую нужно доказать, и не ведущие к ней - заменены на специальные ячейки типа `PrunnedBranch`, которые вместо данных - хранят их хеш. Таким образом, посчитав корневой хеш всего дерева, у нас должен получиться тот самый хеш из данных ячейки типа merkle proof, который должен совпасть с заранее известным нам хешом, например, блока. Такой подход позволяет нам доказать, например, наличие транзакции в блоке, при условии, что тот, кому мы доказываем это - знает хеш блока.

##### Пример пруфа для getAccountState
Объект является пруфом для структуры [ShardStateUnsplit](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L399). Но вместо всего блока - содержит только наш запрошенный аккаунт - `EQCVRJ-RqeZWcDqgTzzcxUIrChFYs0SyKGUvye9kGOuEWndQ`. Все ветки не связанные с ним и не ведущие к нему заменены на `Prunned`, а те что ведут к нему - являются обычными `Ordinary` ячейками и содержат реальные данные блока.

Символом `*` помечены специальные ячейки, корневая имеет тип 0x03 - MerkleProof, остальные 0x01 - Pruned.
<details>
  <summary><b>Развернуть пример</b></summary>
  
```
280[03E42E9D59FE3B0900D185EA76AA5F64C6FA0F0C723FFD3CCA5F22B21354064C540219]* -> {
  362[9023AFE2FFFFFF110000000000000000000000000001E9AA7F0000000163C1322C00001F522060E3C10194CD420_] -> {
    288[0101CA91D55F8B903536211B4FB74EA37FD7930C3774D76FE64B45BCB1FE0E1E26380002]*,
    75[8209C398C55C7BDDF32_] -> {
      76[0104E1CC62AE3DEEF99_] -> {
        288[01011D8C7CB5BFB085D47FB85883C71A7A1DC5B4856A2B9C27DE05F63E994D4A557A0216]*,
        76[0101A1AB8F37E15D045_] -> {
          76[01010A762B5E3DB9BB3_] -> {
            76[01007896888C1BF9AFF_] -> {
              288[0101658DA3EABE6B20FA399239F51BF78EB7DB0D8AAE0572E682B986268E041F71590036]*,
              68[00F45169A7CA17BB3_] -> {
                68[00E1E486CCD6F2D24_] -> {
                  288[0101F2B70E32B323C3E1817C1780D7A2A49D08BA63B9B9D024B3C713AB936ECA3FD40028]*,
                  68[00E17E05DDDEF5882_] -> {
                    68[00E0EEBE3D59F5624_] -> {
                      288[0101447569C499386266F623BB34E72EF718158ACC2E76ED42F88C6D842D503CDF900020]*,
                      68[00E0B73C712EA7A3C_] -> {
                        68[00E026EB80E87B804_] -> {
                          288[01015DB34588AAB7AF2A5034E0C26168E51FAED17544C56405B0F80CD967A4EE5BD9001B]*,
                          68[00E023322BD069152_] -> {
                            68[00E0220B791895CE8_] -> {
                              68[00E0208438C7659B4_] -> {
                                68[00E0201CBDC2FA57E_] -> {
                                  288[010132D3C686F68E1E93D14D3DB79EE7D88FB582C87B35BB5B553D82AA25D09196130014]*,
                                  60[00C047CA5AA50C4_] -> {
                                    52[00BD53FE01932_] -> {
                                      52[00A0AC70E85E2_] -> {
                                        288[010178A421C00E63D315BAEB8086668F7D45632ACD4EF17B8A33884C1499856572E2000F]*,
                                        44[00894655318_] -> {
                                          44[0087FCFA2A6_] -> {
                                            44[00805E35524_] -> {
                                              288[010147E47997E4E71FC094644C589682DE513FB731A6B8E841EB2AB68DF4EF1BD92B0002]*,
                                              53[E0A04027728E60] -> {
                                                570[B991A9E656703AA04F3CDCC5422B0A1158B344B228652FC9EF6418EB845A001C09B2869397E6EAB3E5EC55401625FB84ED36424BB6EB0F344BC13953F74CC340000716D4326F30C_] -> {
                                                  288[0101BC95189D8B489591E527DFE6FC8787A3A2D382C1DCB6AA9153D7A538F9FFA3B90007]*
                                                },
                                                288[0101D3016CE07C229433613139886E3B2578C61C374FB3AB0C1889666D8588B5B0090008]*
                                              }
                                            },
                                            288[0101641854A854E746372DB4110640D8AA4AFFDCA2CE5F110980D7EB7CF4159652A7000B]*
                                          },
                                          288[01014C4C14F8F5E62FE6145EFBB10255D3F82082E492D8A8E6F17AC610C70A5B06A8000C]*
                                        }
                                      },
                                      288[0101DBC646BFEA5C4D04E6D76C3DCD521BECDD2062F493B9727D3466E0F5272F2FC90010]*
                                    },
                                    288[01017B7679129CCE779E8ADA70B51D8FD2A5B1FD8180E98FE35944399D3E583E49140013]*
                                  }
                                },
                                288[0101853D4CD469D08FBE03DA3DDA48568763B19BC5A711BB7887E9D7700D24314275001C]*
                              },
                              288[010156C1849FDD43EFEAD3B9FCFB246A5FC40D0D9AA4F4B0C877A9AEB5C4AFFF84850018]*
                            },
                            288[010157B0C46BA1C3726890924C2D7F3F4B755BF7B24363CC325B37C9F956DF569AD4001A]*
                          }
                        },
                        288[01014FD09DCDD654FD4C3758B75C755893092B5D7845356647C3CB5028AE4CF75E26001C]*
                      }
                    },
                    288[0101D3E6EF78A7FB3E14C066534F5172078FCC4382D2AD1512470C25E1504CB3BFE50025]*
                  }
                },
                288[0101CDF101C69F77450BAC9532A70318AB7F0DD902A4A3EE2CE8CDF5262547B5CB68002A]*,
                288[0101AAED7CCC3904836F362AE06EB234B71D64E02EB4BA6D6B7869197A9ED5C4B0B80001]*
              },
              288[0101AAED7CCC3904836F362AE06EB234B71D64E02EB4BA6D6B7869197A9ED5C4B0B80001]*
            },
            288[0101EE15491F5B3FFB71A0C759A476F99A84D7A1B5CF8A47B1DE5D1039882CD723710065]*,
            288[0101EC90A44EEE02BED840C10E88351163EE9E3613EB9DBE8DA760783DA449714E280001]*
          },
          288[01011AAC6151A41BCC89EEE855D29D087ACDD30BC4F2069B246BE0C69E3C228C55110067]*,
          288[0101BB06F3506745C5F6A6239D132A70B38439CB60FF95F62E45261BA12E844E889B0001]*
        },
        288[0101B3E9649D10CCB379368E81A3A7E8E49C8EB53F6ACC69B0BA2FFA80082F70EE390001]*
      },
      288[0101B3E9649D10CCB379368E81A3A7E8E49C8EB53F6ACC69B0BA2FFA80082F70EE390001]*
    },
    868[0000000000000000FFFFFFFFFFFFFFFF8270E631571EF77CCBBA0EF6D39D6343100001F5220425F440194CD434074F1CF0713270443B84FEA5C35142771F8C6895A2B36117743862354F4EBE767D25641E7D9D005816619EB807E1C18C943B86598038ADD33B65980DCB14EF2_] -> {
      288[0101B3E9649D10CCB379368E81A3A7E8E49C8EB53F6ACC69B0BA2FFA80082F70EE390001]*
    }
  }
}
```
  
</details>

