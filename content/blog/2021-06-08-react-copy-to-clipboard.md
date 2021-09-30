---
title: "npm react-copy-to-clipboard light-toast"
author: [acdlite, bvaughn, abernathyca, gaearon, rachelnabors, rickhanlonii, sebmarkbage, sethwebster]
---

[react-copy-to-clipboard](https://www.npmjs.com/package/react-copy-to-clipboard)
[light-toast](https://www.npmjs.com/package/light-toast)

```js
import { CopyToClipboard } from 'react-copy-to-clipboard';
import styled from 'styled-components';
import Toast from 'light-toast';
import FileCopyIcon from '@material-ui/icons/FileCopy';

const ExContainer = styled.div`
  width: 100%;
  display: flex;
  flex-direction: row;
  margin-top: 15px;
`;

<ExContainer>
    <Typography
    style={{
        color: '#ffffff',
        fontSize: 12,
        marginTop: 8,
    }}
    >
        {address}
    </Typography>
    <CopyToClipboard
    onCopy={() => Toast.info('Copyed', 1000, () => {})}
    text={address}
    >
        <FileCopyIcon style={{ color: '#ffffff' }} />
    </CopyToClipboard>
</ExContainer>
```

