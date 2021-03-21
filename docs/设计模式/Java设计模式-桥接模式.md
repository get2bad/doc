# 桥接模式

![](http://image.tinx.top/img20210316113730.png)

见图知意，这个桥接模式就是将抽象与实现分离，使之可以独立变化，它是利用组合关系代替继承关系来实现的，从而降低了重选ing和实现这两个可变维度的耦合度，这种类型的设计模式属于 **结构型模式**



## UML图

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAosAAAEBCAIAAACfS3hjAAAgAElEQVR4nO2dLYzdxtvFjaL+WeALl1gKXBhgEFhYs8KVTAoLDMqCrMISS4UBBoWRUaFhyUiBlVGYAwsMAucFp/fpk/HYO/fD9qzv+YHVXV9/PHd8Zs58eZxYQgghhMRHsncAhBBCCPFAhyaEEEJihA5NCCGExAgdmhBCCIkROjQhhBASI3RoQgghJEbo0IQQQkiM0KEJIYSQGKFDE0IIITFChyaEEEJihA5NCCGExAgdmhBCCIkROjQhhBASI3RoQgghJEbo0IQQQkiM0KEJIYSQGKFDE0IIITFChyaEEEJihA5NCCGExAgdmhBCCIkROjQhhBASI1s79OPjY0JIkiRJ8vj4uLH8CCHkBbG1QycJW+3kXygGQghZgA5NdoNiIISQBejQZDcoBkIIWYAOTXaDYiCEkAXo0GQ3KAZCCFmADk12g2IghJAF6ND/MgxD3/d6S9u2ewWzzDAMe4dwG6IVAyGExAAd+l+yLCvLUv4tyzLLsh3jWSDLsjzP947iBkQrhgjhQgLkfuBKCQId2lprh2HIskxbsjHmLIfuus4Yc/PAiqKYbmzbtuu6m19re+IUQ5wwrcj9QLULdGhrrW2apqqqNE1ly7kOXRTF1KGv7I5eiGEcx2cPD9lnX+IUQ5wwrcj9QLULdGhrrS2KYhiGNE2lbQp3LMsyz/Msy2SIummaPM+xsaoq2ZKmKTqfm6bBCbMswwnxLUa127bNT2BPCcDZiEukaYrt2Nj3Pc6GSwvok8/zXLbjcJw2y7I4h67jFEOcMK3I/UC1C3Roa62FBRZFIUPRxpg0TdEsrutaPFIatV3XaZvMssxpQxtjkiTBCcuyhEPLOXF+uTpOpTfa+Ta0VA5AWZbyb1EU8Hgdv94hKuIUQ5wwrcj9QLULdOh/B6GrqkJzExu1O2rjTNO0LEtn1redcWhtt6Dve1g72sdyTmnj6pMEOrQ4sbW26zocoo/F5QJSYmsiFEO0MK3I/UC1C3Ro2zSNbndi45xDD8OAEWvHkgMdOs9z9KjLt97dpjFopg49vSgd+mAwrcj9QLULdOh/B6HxWfdCTx0aLWBsrOta2544tB7Jdqy36zpdA9BtaGmUd13njIXLdn2tuTY0BsUtHfpwMK2stZVCT+MIRLqv4hz0IQLVLty7Q4/jmGWZTHuWp6LhoHDuoihknFhmXTVNox+Fwhj2MAyycc6hcS00xLEz5qMNwzAMg37QGZPXhmGo63rBoZumwU9Adz3Mvm1bOvSRYFqBhT6nQDZz6HEc43+eIk6oduHeHRrPWaE+jip2VVXIWsYYDE5Lbb3ve+yf53ld186pyrKESctpgTbXtm3h5X3f13Uty5bB7+VwZ38JYBxH3YyQ3bquk9Nii1xaDonwEerYxBAzTCvwghy6rus11ki4B6h24d4dmuwIxRAO0wo4Dl2faJqm7/uqqoZhQFXbGIOvnFqv16H1nqjUtm2Ls+Hksicq31IVruu6LMtxHHEGbEQAmFXqVI5xuGzBnqhb43OcD0ZuDNUu0KHJblAM4TCtgOPQTdNgsQHYIQaMMHSVZVnbtthB257j0BgbatsWA0OwXqwlUNe1nFy2o2WcZZmeXprnOfqxMG6FHjjsY4yRvm6Mlxlj5MFO7IkBL3TOHWNB3yuh2gU6NNkNiiEcphWYWzMAUzX1vzKEpFfysROHhhPjc1VVsNjp2YwxMg3TTmZ6woOd2JznOzBfRP7V9Qa9BhHb0JZqV9ChyW5QDOEwrUC4Q4s7OjMlHYdGa1sWCoQHe8+GjmvZUzu0NzbHocX+5VupGejHMYil2hV0aLIbFEM4TCtwgUOXZelYo+PQ0/fMzjm0fv2dEO7Qugdb+sktHXoC1S7QocluUAzhMK3ABb3caZrqqV6OQ+umbdd1aNd6HVqvZzCOoxy14NC4Lny973u98JHekw7tQLULdGiyGxRDOEwray1asZgUBvNDDzamg8FrsTaAvEhGL7YvK5bgK5nAhY5rzDKzpzUS5Gxycnt6RQ32xNwxzCaDryM2qQ1g6pmspmBPCwohBlngSGZ9X7AGy1Gh2gU6NNkNiiEcptVZwALPmnUVvvN0Wf6zuPLwe4BqF7ZOiFevXiWEJEmSJK9evdpYfi+XhGVWMHh/TFEUXNLrhUK1C2xDk92gGMJhWpH7gWoX6NBkNyiGcJhW5H6g2gU6NNkNiiEcphW5H6h2IUaHvmZVnWEYIpwSiTdY7B1FdDAfhsO0IvcD1S5E59B4tuGykw/DcP2rbxzGcez7Hg9dXDAJUw7n845TmA/DYVqR+4FqF6JzaHv10wi3dWhwpcVu5tARvmJyAebDcJhW5H6g2oUYHfrKtePDHTq8KhCbQ8/1mevVDS87w5YwH4bDtCL3A9UuxOXQXddh1R7ZgjfP4L1ydV1j0R/pdsYaQ86yuoEObYxJkiSwNqAtVq6e5zlia9t2HEf8W52YOxwgeNkTyxi1bYsViPQKwM6e9vRKWmyRlX77vpc38SEebMdrbvXhSGR5A65O7Y1hPgyHCwmQ+4ErJQj7ODQcSCNOKWvhAnnbOawIb1e133qe4zHhbejwdq1jsbiiBCOxyctqyrLUHc7O4VjqD5+bpsHUNpwEC/TLv9498X5ZbJQYgPNy2WEYZE/92RiT5zkOlJi3J6FDB8O0IvcD1S7E1Ya2Pod2XFD/6+wDNhiH9gajwxCj9R7u7InDvQ7t3VOffNmhpX4jYcipwvvD14P5MBymFbkfqHaBDh3EzR0aM8MFO+/Q0z3Pcmjvt054e/GsGD58+PDu3bsPHz5sEk7UsMwi9wPVLhzBofM818PJ4Q4dPlvqXIeu69p54Z0+3Pl2waG9e4Y4NDY2TaM7sXUbOnKH/vDhw8PDw9PTU9d1T09PDw8PB/Ppp6enn3766c8//wzcn2UWuR+odiF2hy7L0hiDeVjyr1Xvf8W8J32GQIdu2zZN00CT1hbrBCP/yvQuPeI7PRzkeY7+56ZpUL3Ar8bsLavcdLpn27bykzHhS06Lbu1hGKRzW8b467qWjcYY77voN8YrBvHmz58/y8bPnz8fzKc/f/78+++/f//9969fv356evrjjz++fv26sD/LLHI/UO1CXA49DEPf98YYeQ4Kr5DDUiTyr7U2z/NxHNu21U9M6cNDJmk7k8C9YMkRnBPXGsdRByOxoTnbdZ0e+p0eLhhjdAAwe7w3Hh+k9jDdU3enO97fdZ2zBYdLgkwTeS8cMXi9WXM8n7bW/vPPPx8+fPjxxx+/++67H3744cOHD1++fJnuxjKL3A9UuxCXQ4eD97Tf5FS3YvrcF1lGxPCsN2sO6dPg48ePT09P//d///f27dvffvtNpwbLLHI/UO3Ci3RoNDF3bwI6TJuzZJkkSc7yZs2Bfdpa+9dff/38888PDw9v3rz55ZdfPn36xDKL3A9Uu/AiHZocgyRJHh4ezvVmzefPn7/77ru110+IhNslPCFRQ7ULdGiyG0mSXNMOvrj9/SJweryZccj9QLULdGiyGxDDBf3VR/XmhVljzDjkfqDaBTo02Q0thkCfPqQ3hzx5xYxD7geqXaBDk92YimHBp4/nzX///fevv/76+Pj48PDw7OolzDjkfqDaha0Tgq/oIcLcG2wcnz6eN1tr3717J/O0Q/ZPWGaRu4FqF9iGJruxLAb4dJIkB/Pmy2DGIfcD1S7QocluUAzhMK3I/UC1C3To/ZE1RPcOZGsohnCYVuR+oNqFF+DQsa3ueVuapoE3d12nX2N1DzAfhsO0IvcD1S5E59Bd1+k3S7Ztq191FRVZlul3O15A3/f6LZDyHqo7gfkwHKYVuR+odmFnh562GquqcixZG3YIa5icfl2V4LxZ6wKmP815beWxYT4Mh2lF7geqXdjZoXULUrYURaGd7yyHbtv25gO64zhO47ye6Wuk7fnVkRcN82E4TCtyP1DtQnQO3bZtXde69zjP87Zty7J0dm6aBhvxzsdxHJumQc+ztG77vq/rGrbXNE2e52hho3u5LEunm7qua+dCbdvmeV4URdu20pKWazm1AQkJVxnHsSzLruvqusYZnJ29XQj3M2WM+TAcLiRA7oe5lRLukH0cummaqqqqqsrzHB8wHazrumEY+r7XTck0TWUulTQ65RD7bc+w1+FkwLgsS3wr5zfGiB/DTbFRB6D3cU6rr1XXtVi4HI56gLNxIdSqqq4c235BJHToYJhW5H6g2oW42tAwLWNMmqay0XFrfNDtbD3Z22t7+mwAbVxjjG4x6/Fv3c0e6ND68KZpEJ6OJ9ChvWPeh4T5MBymFbkfqHYhLoeW5maWZeKRXoe2p17rPM+1m3ptbzq4K8Ys7muMmZs0HujQ3mb3gkN7T1uW5ZWzz14QzIfhMK2uAQNMzjATiRaqXYjIoYdhkHHZqqrkszY2MVGxsWEY9A4hDt11nVQFtE1qh9bt8pXa0IFbDgzzYTjxp9W+DwoaY+Y6n8qyRHYehkEy/o4sJNT99J8tE7/aNyOi56Fl7Q5rrTFGRpfF+bquE6fUY8/a1WT6la4vTx1aDm/bVuaayTi0tVYPBsuka8f7F8ah9Xj5gkM7fdr3tmgJ82E4kafVMAxJklw2yXEcx77vsyxrmuayDqS+76czSUHbtjpP6Uy6CwsJ5Ux/uWciV/uWxOLQeKJJbBXGiexqjNE+CpqmkY1OnbSua5lN3fe97KYzP6aq4di6ruUrzOWe9obhJJKvxnGU0+oMj4noTdOgzo4fhQICs8enpYOuasRQwd8S5sNwloeHNsbro1eOzkyfjDgLTDidbo+wm2ohoXaPLRJYMgixOPQ9Awu/nyncAsUQzgUOfdZyueE713XttdIre7kDHXouTq9Do23tbCyKIjzUNbruF85JhwYsGQQ6NNkNiiGcZYdGj452KSyXiy3O9EN0CEufkKwZgA/ilPJIpPRmoesoz3O95ICcYfrEBI7Fzji8KAp0cZVl6Xindmjp9Ma1pFsLB8qPml7LCcDbpx2+5EBVVdMf5YBVipEUuAuoQ0zviJ1JKIyOo7ONDg1YMgh0aLIbFEM4SCvxtizL8GEYBr04nf4sEzjGcZTPeZ5LM9SZgykTONDI864ZYOcdzjEevci8NFv14/6OG3nnXcq1sLOenoJVjHRUU4eee6AxfIwgpOte1kFqmgYf5u4IcBJq7o7cMywZBDo02Q2KIZyFNrQzv0GbsWyUz44riwN5HzXE1MWiKC5waH1CmeO5MHHS++zi1KF1JM5DHDdvQweiH9yQS3jvCNAJNQyD/pYODVgyCHRoshsUQzjLDu19Lt/r0HBlMPc0v2yRJfYucOhzlwdYw6GvH4cOBM1o/bDoQtrqhHKWYaBDA5YMAh16LWRh8L0DiZf7EcP1OGmlnyCSNhzwtqFl49zCeY43YBAan+cc2tH2QhsaDzjYmzp03/fOcr87zuUuikIHM3dHwEJC0aEBSwaBDr0KUljoR7ni5KwZv7flTsRwE5bTSgZ9db9umqbQnhikPb3cxVqL5wzlDI6L6IWAMFdLziBd1s5DiY7x4IUx9tt1QvSjjHr81X7r0OM44upYSF/+xRPDOMqZ/jbn0Nc8D43x/pA9nbfa25k7AqYD9vI0R5ZlXLTEsmRQ0KG/YRgGKdcuxsmTu7/yGdN6p9vHccTE2u1DApGLISqeTSssG6D7b2FmMn1JGIZBz8TGgiHGmL7vtWWi29bbXMYyA/Lvwhmct8Fi1X0cYoyRV8DJ4TJpHLthn2EY5F9MLHcqBzgcZ5imjJgl1iRYTkYnocJHrKeXnt6RuYSSX7TvumzxwJJB2Nmh911Cy5sfrn8uebqE2e4rhc39qLkFTbeB+TCcc9NKz98+DM787bMOtHuXNiQclgxCROtyb4xTu78V3rc+Rzu8RId+KZybVmj/HaxNhobyjuMyZBtYMghxOfQwDHjiU3qxsIY26s7Ozhgek9ahLLxgT6+90l1bek90/eGl0dOFF6YtDwzdVaeVRMdxxAredV0jNr2zd6ZouEPnef5sK0EvjIDVT7F9mnr29JoQJwAsR4oBMDr0i4BpRe4Hql3Yx6FluaI8z/EBfiNDtvolGWmayiiUOA2mkFg1qQTAd/UOVy68oAeV5VR6SSPH/LxmrB88XcZZ6HsOqUboX+RNPet7saYE6ey5McyH4TCtyP1AtQsRtaH1jFP73EMjzkZptk5X6cNXxhinFX7Bwgsh75T0OvTNe7mlh0A6BuZSz04ceu7lm9vDfBgO04rcD1S7EJFDn7vwQntCmuDW54VizNssvOA97c0dGn3sXddJ03xhkQTHofVzKXTolwLTitwPVLsQkUOf1Yaem6c6nUc9115cqQ09XcZoujDvTcDKzPLvWW1oOvSLg2lF7geqXYjreWg9kir9t+Iu2m71Egr6UaKpQ+ul9iv1kmmZdL288IIeh5ZTLTj0dItMMQsBA/MhexZF4VQFvKlnfePQ8u/0FUNbwnwYzuvXr5PN+d///rf9RckL5YZqef369d4ZLhbicmjMspbX1QGsD4y3s+mdMd1M9sRT/9iip2XJblhESX+F9TrEPufO0LYtWqjoS8dEaASDieLO3C69iJIx5qwHrKdnm2P6LI039bqukx8lYwFIiqZp+r7fcQ2jZTGQfXn//v3j4+PPP/+8dyDkBfD+/fu3b9/+9NNPewdyNOJyaC/RPky8gDEGzrdjH3L80KGj5e+//358fPznn3/evHnz+fPnvcMhUQO1fP369e3bt3/99dfe4RyK2B0arWeuVXtI6NDR8u7dO2S6jx8//vDDD3uHQ6JG1PLp06e3b99+/fp174iOQ+wOTQ4MxRAnHz58eHp6kn/ZMCILOGr55Zdffv311x3jORh0aLIbFEOEfPny5fHx8cuXL7Ll06dPj4+PO4ZEomWqlq9fv3Jk5IbQocluUAwR8vT09OHDB2fjjz/++Mcff+wRDokar1r+/PPP77//fo9wDggdmuwGxRAbXde9e/duuv3Lly8PDw8cXySaObVYVulux9ZF5C6PdUYInzRN+NRjZHz9+vXx8fHvv//2fsvxRaJZVsuXL1/evHnzzz//bBzV8WAjZgd++eUXPmlKYuP9+/fv37+f+/br168PDw96xJHcM8tqsdb+/vvvfDz6eujQW/PXX3+9ffvWWvv4+Pjp06e9wyHEWvVI68I+LHMJCFGL5VMAt4AOvSm6a0ismpDdkUdal2G1kthgtfDx6OuhQ2+KM5jnnQlJyMY4j7QuwGm6JFwtltMXroYOvR3TRjPnU5DdmT7SuswPP/zw8ePHVUMi0XKuWvh49JXQoTdiburjb7/9xiljZEfO7chh1+U9c0G3H/tdroEOvRELvT0c2yN7sfBI6wI///zzb7/9tkY8JGYuU4vl49FXQIfeguVJYZwyRvbi7du3lz3L/ubNm71jJ1tzsVoomIuhQ6/O8qP9gFPGSIQkXPSNBEO1rAHTdHVCZjNyyhiJEJa5JByqZQ2YpusS3oPNKWMkNljmknColjVgmq5ISP+2hlPGSFSwzCXhUC1rwDRdkXOf1ueUMRIVLHNJOFTLGjBN1+Iyu+WUMRIPLHNJOFTLGjBN14LPsZCXDstcEg7VsgZM032gmkn8UKUkHKplDZim+0A1k/ihSkk4VMsaME33gWom8UOVknColjVgmu4D1Uzihyol4VAta8A03QeqmcQPVUrCoVrWgGm6D1QziR+qlIRDtawB03QfqGYSP1QpCYdqWQOm6T5QzSR+qFISDtWyBkzTfaCaSfxQpSQcqmUNmKb7QDWT+KFKSThUyxowTfeBaibxQ5WScKiWNWCa7gPVTOKHKiXhUC1rwDTdB6qZxA9VSsKhWtaAaboPVDOJH6qUhEO1rAHTdB+oZhI/VCkJh2pZA6bpPlDNJH6oUhIO1bIGTNN9oJpJ/FClJByqZQ2YpvtANZP4oUpJOFTLGjBN94FqJvFDlZJwqJY1YJruA9VM4ocqJeFQLWvANN0HqpnED1VKwqFa1oBpug9UM4kfqpSEQ7WsAdN0H6hmEj9UKQmHalkDpuk+UM0kfqhSEg7VsgZM032gmkn8UKUkHKplDZim+0A1k/ihSkk4VMsabJ2mj4+PCSFJkiTJ4+PjxvILh0IlyTkSpWDIGgXa1g6dsJ5FTsQshphjI5sRLgMKhqyhATo02Y2YxRBzbGQz6NAkHDo0ORQxiyHm2Mhm0KFJOHRocihiFkPMsZHNoEOTcOjQ5FDELIaYYyObQYcm4dChyaGIWQwxx0Y2gw5NwqFD/0elqOs65JC2bbG/bBmGIc/zcRyviaRpmqIo9GnLspR/x3F0vo0KHer2xFyoXRZbVVV5nu+bqs/SdZ3OPhfov6qqoijSNF0jvJvTtm2e55cdu6pDN02T53lRFGVZnnvslmi1NE1z7uFN05RlmaapMWaN8G5F0zT6lw7DcO4Z6ND/YYyRAqIoimdzYF3XRVFYa/M8l51xkr7vrwwmyzJdIqdpquNxqgXP0nXdlfGEnxYFxBqXC+FgDp1lWdu29ltxbky4eBzRnsuWvxGpejFZll0c6noOLcY8jmOWZbu4V7harr/dmzn0OI4XN7okCzdNk6bpuSZNh/4PrZiu655Vj+SBYRguqBw9e3Ip7IwxeZ7reM516Ivr+wv0fR9hq+5IDo3Ws/53+zL3rLv8ghz6yhwBh76sIr6SQztFFno1zovsFoQn7Aty6LquL76QriqlaRrYOyvQof9DKwb1HfkKnRWSIaH+NE3R9yjVxrqu0VknB2LPvu9xBm3kwzBMe3hkoy7scAldIsChcXKn0ooYjDHYXtc1uoN096OOs2kaLRp85ZwTUclu4zhWVZVlWZ7n0+1zP8rZrW1bb/xXciSHTtN0rgMQt0nkhLs8jmNd104RgESW8yzcekeQc3dZzlPXtVMx1aKVu4w467rGmRek6y2ynT3lnF3XtW27EP9c1pPu9On4lPMzsQNapRI/4kTKnFvagpUcuiiKub4rRwMLhZL9Vlo6RyN99C1zSkVvUaOvuHy7vXcWBxpjvHrzOrTeE4fjbM/GLx3R07SaFvVyuLfw1zlLHHocxwtqdXTo/4BijDGwZ7lJyIrGmCzLsHEcR/nXGCNahOi17NDdlOc57lmWZdjeti3unN7Y9z26RHASKT5QLdUlAkRT13XbtmmaikryPG/b1hgjA9V938vvEjVLnEVRyLWGYZDd9Dg3qgtIFgnVGFOWZVmWxhitOSSLLvjQAYBcl2UZck5ZltgN8d+wB+JgDu2tuUNOSGrdB4501iU1PsvNsjO33s4I0nuXh2HAdXGIvvvOrc+yrCiKuq7TNMWN1leHMPRw6dShy7LEnvKjoBycra7rPM+RKabxz2U9UbLOEV6V4ufLr0iSBPvDM5wejnBWcui5DgxMaglJGSl/cCMkEbAzutDlBk1LRW9RY303ETi323tnEaouFXVZ4WSQqTJRpmE4EnEuxK+V+WxRjxISP2o5Z2VZBne/rIeJDv0fUIzjeZh5oXeQ/b0jPdNSRopRa60oTBdtogYpbuy3+Q0BlGUpkeiWui4p0jQVDemRtmnTxBiTJAl+JkLCHDd8i9wix2K3vu9DxsKnxbSuZiJsaZron38TDu/QehIQilTZGbdeFNj3vdx3SXnru/V2RpDWd5dhV/KtLnOntx6FNYJ0/nUitJO8g+JVpwbaN7gEvpJ/vfF7s549uY7+UV6V2lNVGAkr50cuCBkI87KlQztpCFe2MymjK0x5nuv0lIyPjQulopMm3pvoPXDhzkqo6M7RZ9MZxKtM52z41xu/VsVU2PpCuq2if9RczoK74yi2oS9HbpVWKrIo5oI500PCHVqPQ+CznBMDzKIe2VPyG2r3sDQ5sy43UTvDZ+wj+VDwOvR0I6quqArgnE4Zqgl0aG+e1MdeOXjpcHiHdio00m8mt0kSWTePNN5b7xWk9d1ltGkkR+hK27kO7fxGJzDpGJCo0HL1luPe+L1Zz/oces45dOZyfiZKhlVn517v0FMXlA45b6Hk7bOZnnmhVHSk5b2Jc7HNObQc4twOJ2CvMr0O7Y3fcWh9Iaeon/o3agbenKWP1a2sQOjQ/yHpq2vHurXncKVDTycHeh26Os0PQm0UJcKcQwNU1nTYIQ6tK4ZyTqcKrAl3aKk20qHP2t9JmaZp0GjWw58iialDOzV90ducQ3tnq3odem4i9GUOPdeomvqonS/HvfGf5dBTlVpf5kItQY66oPtnJYfW/V7WWoz9T10Q1jJXKOlxVt0KnDr0XKk4dWhvNdFe5NDoMNfXchx6qsw5h57Gf5ZDOxVTyOBZh/ZW+JahQ/+HTl9JdO3WeBBZ9r/GoXWHdnWaraA7MKUdoKUgJYJWmIwZ67JjrhyUo6Zx6qtDSZC70+ep1SbTIhYGI/UvlVDp0CE4Fgsl6D5Y3Vk3dWhdpbNKSN5yxCtI67vLugHRdZ2uLpzr0M4PnAami2DEP1eOe+NfcGhJIhzlVan1Fak6CyxMzlpgJYeWeSQSm1TddL0fHuxNGf1z9FTwaQ5dKBWnRc30JoILermdKpHj0F5leh3aG/+yQ+O6MuQsh+s2zLMOfcGTqHTo/0AXMe4EegihZlRO0UGBnC/TXJ2VQ+Q5+uo0xw9SkDIuTVO5xzjcWVsAG9EdhNZSepqg33UdunFwdXRH624T6AMhYcqY/mno0kFU0zglJKgck10RGIQ7DRXS1Lnamf0reyJU2VNmf4zjiB+VX73Gi3Akh7bfas9pSehbr3WiZSy9KTLw4b31dl6Q07tsTwsA6ABQoOO2VlWF50dFTtlpkg7sTVTqNC8kMCmFJctI/Lg0einx6xDDNP65rAcko+kfpVVqT00l+UXOOcdxRD4992GE9Z6HniaXVZM3ZeNCyjh3di5HW1+pKNt1UTMXldzusiyX76xcXSsT86jT0xRruboTP36pnA2HzJXqKPS6rsOdzdTUBEw9c+bP4oSiYcLiHgQAAA8PSURBVG/OkrljSP9V63NnnPPmZ3zmepsUytcvQjLFO4h11oXCzxA4YDZ39bnDA0978+fF5ziYQ4Nrbujc4V7OusuX3VNpqYRHdX38gTtvo9K1V/30Jtf1pUr4ta4v1jSwwCvv7BxXluprmIIDHZocipjFEHNsm6Gb+PcJ1+UOB31sRVHcqo/txUGHJociZjHEHBvZDDo0CYcOTQ5FzGKIOTayGXRoEg4dmhyKmMUQc2xkM+jQJBw6NDkUMYsh5tjIZtChSTh0aHIoYhZDzLGRzaBDk3Do0ORQxCyGmGMjm0GHJuHQocmhiFkMMcdGNoMOTcI5gkO/evUqISRJkiR59erVxvILh0IlyTkSpWDIGgUa29BkN2IWQ8yxkc0IlwEFQ9bQAB2a7EbMYog5NrIZdGgSDh2aHIqYxRBzbGQz6NAkHDo0ORQxiyHm2Mhm0KFJOHRocihiFkPMsZHNoEOTcOjQ5FDELIaYYyObQYcm4dChyaGIWQwxx0Y2gw5NwqFDk0MRsxhijo1sBh2ahEOHJociZjHEHBvZDDo0CYcOTQ5FzGKIOTayGXRoEg4dmhyKmMUQc2xkM+jQJBw6NDkUMYsh5tjIZtChSTh0aHIoYhZDzLGRzaBDk3Do0ORQxCyGmGMjm0GHJuHQocmhiFkMMcdGNoMOTcKhQ5NDEbMYYo6NbAYdmoRDhyaHImYxxBwb2Qw6NAmHDk0ORcxiiDk2shl0aBIOHZocipjFEHNsZDPo0CQcOjQ5FDGLIebYyGbQoUk4dGhyKGIWQ8yxkc2gQ5NwjuDQr169SghJkiRJXr16tbH8wqFQSXKORCkYskaBxjY02Y2YxRBzbGQzwmVAwZA1NECHJrsRsxhijo1sBh2ahEOHJhvRtq21tuu6Va8Ssxhiji1+ttHPBtChN8AYMwyDPcnm5UKHDgI3+0qcwsU55zAM4zhef5U1uP7nN00jP78oimsDmifmQi3m2MBNdH4xxpg5A95MP4EsJNSzdYgjOfSOghmGoWka71d939d1jc9t28Zs0s8W+wd36L7v0zRFzs/zPORWlWVpjKmqSnYehiFJEmPMNUEaY/I811uyLNPnLIriykusRNM0aZpec4ZhGMqylH/7vq+q6uq4/MRcqMUcG8iy7OJb0/d9WZZlWfZ9f/Hh3qtvqZ8QFgqEaTafchiHvqZk6Pu+aZosy/q+v6BlMgyDMWbu6s4t0OLZhaIovKoYxxF2s3DsAR3ayb1yF4dhyLJs+VR1XSO9+r7XdeHLCh0HfZOGYaiqSodaVdVZDr1GxXChTnrNaYuicOra6zWDYi7UloW6MV79XNkkciR9q8O31E8gCzliJYfeVy1zHQPXlAzGmGcL5GW8Dt00jVOO7V6ls/M569li/44c2s7cUedYb3rdpDNHZ922bYdh0FvOcmi08q8PSTOO41zhEvLzFyrC09NOc9GtuGeHDm+LrKEfG+zQc3HOHX6lftboiV04574OvdJI2VyV6JqSYSWH9qb/szdFuEljbMqC5unQ/97Fuq6luwMlVFVVsmUcx7Zti6Ko67ptW7lPGNLQUuj7PsuypmmqqiqKQrdF0E2nEx0ddFVVNU3j+LG1Vgu0qqq6rp2o7KnYqqpKDm/btixLXFrqtm3b5nlujGmapixL2VlC0udE8Hpj13VVVWVZ5gzbdF1XlqWTkdq2lahktzzPJU6db9F7ab9lJYewL9mhpZu3rmvITyutLMsFpUGleZ7jgwygiM6hQBzr1Q/074y82JP8kCmwG/IIQnLurJbEOI6QU1EUWmz4MNWknXHoK/VTVdWz9fK+72UIrO/7oigk/Z07Yn0Fgp3P5l5u5dDOrbHzaSv5XU4SXlygOzpNU5QM4jTXlwyOQzdNA6nkeV7XNTpOFrKAXceh0zRdrvyhaxrpg+wgmUh+u4yC21NSO7cPOQg/814cWoqMLMvwAWqAtvI81+kuN2wYBl09rGZqNI4UdEEmp8rzXOS7vNGe8pu+PVrWUkYYY/QOcri3hNL5Cj9WOu311buuk5+MXDpNFo2TkXSK6c+yjxObN9SQ4brLiN+h54SqE0Q+a6VJCs+JCmWZPfXQ2G+bPnrPOYdzHFpis6qrWRdAzk10LBbfyrXkX4kKdYW5wxdCPUs/Ia2iuq6RdHr+kfeOAKdAmLsjXs5yaHi/VgtSw3trvGmLyjc26r6HwOJi4UddXzLow40xiMeRjTcLgHCHnlY95whRC6px+KwrPZK2Th+PI2xUbuSre3FoofK1oYuikCRzhKJvc6BDe7OuU1b2fe+MfOtqQVVVyEISib603FH0hE/rdHMOPd0TJY6u9S/U2kLyYVVVelBK24Y3NrahhQWhSvEEZP6IdxxkqjT5PL0o2kBOcyfQofUhUtBroV7m0NN9vIdL/BvoR4acdCnhvSNAFwhz2XyOm7ShvbfGm7b6fumaTWBxoU/lcGXJMHVor04WhgLXaEMHgsj7vpfct1CncYTtNMzo0Kk9dWRhC/Jef0In0DUOjbJSn9ORoOwpTRD04Uwv7dxRTHrU5cWcQzvBo2iexnmlQ3ub3XP50HtajkPbxU4RrxnLxqnSwDSd0SM9/TbQob2HbOzQ3t+1hn7QDpYA5u4I0AXCXDaf4yYO7b01cw7dNE2v8P4iO1NcLPyoK0uGF+3QdV3rzonpJRaErXPZPTq0k3XlLmKUzn7bR2GtvaFDy0Zvs0YP9sjGOYdGbVTXSfVzLFrr0r02DX6ufqCTSF9C9tFdPdPhIj3K8mxN2XIu94kFoTrTBmXEzrvRqzQ7KSMW2nZe/djFNrTI5oYO7eTEOYfeRj8YB5XsMHdHwHTYS++5fKHLHNop1ry3xpu2Tn5fcOiFX6Fzt2y8smS4wKGX74KEcc1c7sCpdpCH0zLWkxUWhK07G+7xaSvNOI5yF5ED8VnaFl3XORNwvE+hTDOk3EgRmdRAh2EQpZZlKWM8WZZ1XedMmcZGq+4iZh/gW90T5fT14SQ62ko9wy0/EydH75bMgBMxNU2jFY9mhDktxwPMZMqljIGhImlP04JkfydUPg8NlmPT1TJJH0y2st+OXXmVZn293CLUuakYjmAch+66DufXd1DnET3+an1tBXuaPST/Qoo4ynl4es6hr9HPdDbTAk42994R757TbL5wlZs8D+29NXNpm+c5QtL5Pby4sKekGIZB/7QrSwbncEcn+t9pFgA3fx4afhH4IKseYLKnGWRyxYV8AXdHSZvn+XJ4B3doPNguojTGSMJhfM6ZdWyMMcY46Y7t8mS99jCcX845DIOWtZwWt1wfIifHFe2p7qan18rVnY1yLadqPA3enqoguLT+yqmaCLKzXEV+vt7NGKP31OngdL2CpmnkcqsuIPByHdqeNOk0izHL2kl/R2lTlQoinmEYnDs7bWrgDM5ubds6LSdRrM5cUwVCEtiID/hbVZVkCufqU6WBi/Wjs9uzTPec3pG5pHay+QK3WrHEe2u8aWtPGVZHe1Zxga/0ta4sGWSMRpIR/2qdyEDPNAvoq08TXNfh9Gp0IYSvM+HVVdd1zuXmkhq/6Nkm+8EdmsQD1+W+IDbdW3MMnPnbZx1ouS73IhenbcxclgUM1+VeOOfNz/jM9SIulMnGxCyGc2NDSyLk8Y8XhDSp9w5kN9Zz6OOl7SGzwFnQocmhiFkMMcdGNuMw63KTDaBDk0MRsxhijo1sBh2ahEOHJociZjHEHBvZDDo0CYcOTTxglq8ze/NFELMYYo6NbAYdmoRDh/4XvZzQxWCW/46u1n/70szL0M/aOw/27cLc6lHTpVFs3IVazLGRzaBDk3AO6NAXO8qVr0KTtWy8znFzvD9Tv6bmMpzFdfVj+LswDEOapnOTOac/NuZC7fXr1wm5e16/fk3BkEDC1RJOXOtyzzF9Tu5Kb1tYnW4NVlrX2ru289q/5WJelkMTQsjuROfQ03WFnCWLAYp7rFbjfDU9XBqa2NlZka46vaHPnF7f6134xtko53Qu5yzDaU+vF7UTjDHTXm69XL6Oedob7239y/qCz+JdzOgspiFNl77Se9KhCSHkLOJyaCxtLevh2dNT8FiC2FlhEX282v+wlqwxRjqxZT1VvNIcy8ZO30ZVVRXWpJVzagvHC8i0r+M9V+b0TnVsx0Wne2Ifx+P70wvn9c8vyxITvvSqzliW3ExesusdRJ+usz0HFiVe3qfv+zRNZbkf8X581ulsT0ntrL6LpMDigtPL0aEJIWSBfRwaplhVFYwWBuk0bRdeyW5nXsM+d0iaprLcsZ2sja5fzKLPiZ11v7Fe4D7LMnEsHDX3drO5oe6Fl6jol8jOvT98zqFv++42/d4O2bhwa856jxAdmhBCFoioDT33ylI749DTz/pdvPrNP07rzel2Fqf0vtw0z3M0351Wr9cI67rGyb1vanPwvv7P+Vcf66SP97W7N5/4JtUm3VGxcNe0Q+s3als6NCGEnMmhHHqu+ehs7/veMWNY3ZxDh5zTnrx8+u1KDj0Mw/TNu/IiuRuC36XH3enQhBCyATs7tLaTkF5u/bLb6bf6zSr6zBcYvFiL86J46fF2zulMZ/M6tPNKwcBe7jmHfvZ3LRP+pr/pU2EX9HJ7p/vRoQkhZIG4Viyp61pmimkzw9QkecEqZnXJ1GsxZozC4rWp8HI90cx5QSwMT9telmW4OsKQnfM8x4tLZeAZF8Vr0mU3eDkOx/i6/grzqnQAmGKmp7/BpPUr6Ou6loZyWZbOPG39alWkUvj07LIsA58p7/veaawjKfBLpVYhM8V0UmOWHPaUt7sLdGhCCFkgLocGVz7RG364s6c8wbXGpVd6TFneeT4MwwZLr2guTmeBDk0IIQvE6NC7gHb5S3znvPQl7B3I2UQrBkIIiQE6NNkNioEQQhagQ5PdoBgIIWQBOjTZDYqBEEIWoEOT3aAYCCFkga2LSL6jjQhrvKyNEEIOAxsxhBBCSIzQoQkhhJAYoUMTQgghMUKHJoQQQmKEDk0IIYTECB2aEEIIiRE6NCGEEBIjdGhCCCEkRujQhBBCSIzQoQkhhJAYoUMTQgghMUKHJoQQQmKEDk0IIYTECB2aEEIIiRE6NCGEEBIjdGhCCCEkRujQhBBCSIzQoQkhhJAYoUMTQgghMUKHJoQQQmKEDk0IIYTEyP8DkgtguiFSAUAAAAAASUVORK5CYII=)

**桥接模式模式角色**

+ 抽象类（Abstraction）：抽象类接口，维护队行为实现（implementation）的引用。

+ 抽象扩展（Refined Abstraction）：是抽象类的子类，实现父类中的业务方法，并通过组合关系调用实现化角色中的业务方法。

+ 实现化接口（Implementor）：定义实现化角色的接口，供扩展抽象化角色调用。

+ 具体实现（Concrete Implementor）：给出实现化角色接口的具体实现。



## 开发中使用桥接模式的例子

### – JDBC驱动程序 – AWT中的Peer架构 – 银行日志管理：

- 格式分类：操作日志、交易日志、异常日志
- 距离分类：本地记录日志、异地记录日志

### – 人力资源系统中的奖金计算模块：

- 奖金分类：个人奖金、团体奖金、激励奖金。
- 部门分类：人事部门、销售部门、研发部门。

### – OA系统中的消息处理：

业务类型：普通消息、加急消息、特急消息

假设有这样一个例子，电脑市场上面要销售各个品牌的电脑，有联想，戴尔，华硕等等。每一个类型的又分有台式机与笔记本，如果你用对象将这个分类表示出来太麻烦了，所以我们可以将这个电脑分为两个属性一个商标，一个是类型。可以将它分为两个维度。



## 场景模拟

![](https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=2689756533,3641015967&fm=26&gp=0.jpg)

我们正在开发一个网站，该网站涉及支付模块，这样现在市面上现有的支付有 支付宝、微信、银联、paypal、QQ等，如果给每一个支付都实现一次不同的魔石，即使是继承也需要开发好多，而且随着最后接入了更多的支付服务或者支付方式，就会呈笛卡尔积那般难以维护



## 没有使用桥接模式的代码示例

### 支付类 + 方法

```java
@Slf4j
public class SimpleResolve {

    public boolean pay(Integer payType, Integer modeType, String tradeId, BigDecimal price) throws Exception {
        // 微信支付
        if(1 == payType){
            if(1 == modeType){
                // 扫码支付
                log.info("{}扫码支付","微信");
            } else if(2 == modeType){
                // 刷脸支付
                log.info("{}刷脸支付","微信");
            } else if(3 == modeType){
                // 付款码支付
                log.info("{}付款码支付","微信");
            } else {
                throw new Exception("未知支付方式");
            }
        } else if(2 == payType){
            // 支付宝支付
            if(1 == modeType){
                // 扫码支付
                log.info("{}扫码支付","支付宝");
            } else if(2 == modeType){
                // 刷脸支付
                log.info("{}刷脸支付","支付宝");
            } else if(3 == modeType){
                // 付款码支付
                log.info("{}付款码支付","支付宝");
            } else {
                throw new Exception("未知支付方式");
            }
        }
        return true;
    }
}
```

### 测试

```java
public class Test {

    @org.junit.Test
    public void test() throws Exception {
        SimpleResolve pay = new SimpleResolve();
        pay.pay(2,1, UUID.randomUUID().toString(),new BigDecimal(4.0));
    }
}
```

### 结果

```console
11:57:48.378 [main] INFO  c.w.java.bridge.simple.SimpleResolve - 支付宝扫码支付
```

### 结论

功能可以实现，但是这一坨都是啥啊？？其实更规范的就是使用一个接口，规范支付行为，然后实现这个支付接口



## 使用桥接模式

### 支付模式接口

```java
public interface IPayMode {

    boolean security(String uId);
}
```

### ⽀付类型桥接抽象类

```java
@Slf4j
public abstract class Pay {

    protected IPayMode payMode;

    public Pay(IPayMode payMode) {
        this.payMode = payMode;
    }

    public abstract String transfer(String uId, String tradeId, BigDecimal
            amount);
}
```

### 具体实现

#### 微信支付

```java
@Slf4j
public class WxPay extends Pay {

    public WxPay(IPayMode payMode) {
        super(payMode);
    }

    @Override
    public String transfer(String uId, String tradeId, BigDecimal amount) {
        boolean security = payMode.security(uId);
        log.info("您的微信风控校验{}",security==true?"正常":"异常");
        if(!security){
            log.info("由于您的微信风控校验异常，本次操作被终止！");
            return "fail";
        }
        log.info("微信转账{}成功,账单编号{}！",amount,tradeId);
        return "success";
    }
}
```

#### 支付宝支付

```java
@Slf4j
public class ZfbPay extends Pay {

    public ZfbPay(IPayMode payMode) {
        super(payMode);
    }

    @Override
    public String transfer(String uId, String tradeId, BigDecimal amount) {
        boolean security = payMode.security(uId);
        log.info("您的支付宝风控校验{}",security==true?"正常":"异常");
        if(!security){
            log.info("由于您的支付宝风控校验异常，本次操作被终止！");
            return "fail";
        }
        log.info("支付宝转账{}成功,账单编号{}！",amount,tradeId);
        return "success";
    }
}
```

#### 刷脸支付

```java
@Slf4j
public class FaceMode implements IPayMode {

    @Override
    public boolean security(String uId) {
        log.info("{}进行了刷脸支付",uId);
        return true;
    }
}
```

#### 指纹支付

```
@Slf4j
public class FingerMode implements IPayMode {

    @Override
    public boolean security(String uId) {
        log.info("{}进行了指纹支付",uId);
        return true;
    }
}
```

#### 付款码支付

```java
@Slf4j
public class PayCodeMode implements IPayMode {

    @Override
    public boolean security(String uId) {
        log.info("{}进行了付款码支付",uId);
        return true;
    }
}
```

### 测试

```java
public class Test {

    @org.junit.Test
    public void test(){
        Pay wx = new WxPay(new PayCodeMode());
        wx.transfer("123", UUID.randomUUID().toString(),new BigDecimal(4.0));

        Pay ali = new ZfbPay(new FaceMode());
        ali.transfer("456",UUID.randomUUID().toString(),new BigDecimal(3.5));
    }
}
```

结果

```console
13:32:47.864 [main] INFO  c.w.j.b.u.paymode.impl.PayCodeMode - 123进行了付款码支付
13:32:47.867 [main] INFO  c.w.j.bridge.upgrade.pay.impl.WxPay - 您的微信风控校验正常
13:32:47.867 [main] INFO  c.w.j.bridge.upgrade.pay.impl.WxPay - 微信转账4成功,账单编号e7a82b18-b094-4706-9efa-25518d2e51fc！
13:32:47.868 [main] INFO  c.w.j.b.u.paymode.impl.FaceMode - 456进行了刷脸支付
13:32:47.868 [main] INFO  c.w.j.bridge.upgrade.pay.impl.ZfbPay - 您的支付宝风控校验正常
13:32:47.868 [main] INFO  c.w.j.bridge.upgrade.pay.impl.ZfbPay - 支付宝转账3.5成功,账单编号4e371a63-3cd5-496f-84b6-b394b8a1975d！
```

怎么样？这样我们可以灵活配置我们要的付款方式 + 支付方式，其中支付方式我们可以使用工厂模式来自动生成，这样会更加灵活，这就是桥接模式的魅力

~



看完了我的小案例，您是不是也是摩拳擦掌也想试一试呢？那就自己假设一个场景，测试一下吧~