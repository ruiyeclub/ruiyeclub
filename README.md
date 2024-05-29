import express from 'express';
import bodyParser from 'body-parser';
import {LAMPORTS_PER_SOL, PublicKey, Transaction, VersionedTransaction} from "@solana/web3.js";
// @ts-ignore
import RaydiumSwap from "./raydiumSwap";
// @ts-ignore
import {ApiResponse} from "./ApiResponse";
import {getAssociatedTokenAddressSync} from "@solana/spl-token";

const app = express();

// const RPC_URL = "https://solana-mainnet.g.alchemy.com/v2/VsPvLISKXigzG0nxVoRZGnFxey8WYYMj"
// const RPC_URL = "https://neat-hidden-sanctuary.solana-mainnet.discover.quiknode.pro/2af5315d336f9ae920028bbb90a73b724dc1bbed/"
const RPC_URL = "https://mainnet.helius-rpc.com/?api-key=1cac6fac-690d-4b61-88ad-816a35005901"
// const RPC_URL = "https://mainnet.helius-rpc.com/?api-key=1cac6fac-690d-4b61-88ad-816a35005901"

// 使用 body-parser 中间件解析 JSON 请求体
app.use(bodyParser.json());
// 使用 body-parser 中间件解析 URL-encoded form 请求体
app.use(bodyParser.urlencoded({extended: true}));

// 定义路由处理程序
app.get('/', (req: any, res: { send: (arg0: string) => void; }) => {
    res.send('Hello World!');
});

/**
 * todo tokenAmount是带小数的！！！
 * eg: 0.0001
 */
app.post('/sol/swap', async (req: { body: any; }, res: { send: (arg0: any) => void; }) => {
    try {
        // 解析请求体中的 JSON 数据
        const data = req.body;
        console.log(data)
        if (isEmpty(data)) {
            res.send(ApiResponse.error(500, '请输入json格式数据'))
            return
        }
        if (data.tokenAmount == null || data.tokenOutAddress == null
            || data.privateKey == null || data.slippage == null) {
            res.send(ApiResponse.error(500, '输入参数错误'));
            return
        }
        // So11111111111111111111111111111111111111112
        let tokenInAddress = data.tokenInAddress
        if (tokenInAddress == undefined || tokenInAddress == '' || tokenInAddress == null) {
            tokenInAddress = 'So11111111111111111111111111111111111111112'
        }
        let priorityFee = data.priorityFee;
        if (data.priorityFee == undefined || data.priorityFee == '') {
            priorityFee = 0.0005
        }
        let rpc = data.rpc;
        if (data.rpc == undefined || data.rpc == '') {
            rpc = RPC_URL
        }
        // 处理数据...
        const result = await swap(rpc, data.tokenAmount, tokenInAddress, data.tokenOutAddress, data.privateKey, data.slippage, priorityFee);
        if ('' == result) {
            return res.send(ApiResponse.error(500, 'Pool info not found'))
        }
        res.send(ApiResponse.success(result, '操作成功'))
    } catch (e) {
        console.log(e)
        res.send(ApiResponse.error(500, '网络异常，请重试'));
    }
});

app.get('/sol/quote', async (req: express.Request, res: { send: (arg0: any) => void; }) => {
    try {
        const queryParams: express.Query = req.query;
        if (isEmpty(queryParams)) {
            res.send(ApiResponse.error(500, '请输入查询参数'))
            return
        }
        // 例如，如果你的URL是 /sol/quote?param1=value1&param2=value2
        // 你可以这样获取参数：
        const tokenInAddress = queryParams.tokenInAddress; // "value1"
        const tokenOutAddress = queryParams.tokenOutAddress; // "value2"
        let slippage = queryParams.slippage;
        if (tokenInAddress == null || tokenOutAddress == null || queryParams.amount == null) {
            res.send(ApiResponse.error(500, '输入参数错误'));
            return
        }
        const amount = Number(queryParams.amount); // "value2"
        if (slippage == null) {
            slippage = 0;
        }
        const raydiumSwap = new RaydiumSwap(RPC_URL, null)
        await raydiumSwap.loadPoolKeys()
        let poolInfo = raydiumSwap.findPoolInfoForTokens(tokenInAddress, tokenOutAddress)
        if (!poolInfo) poolInfo = await raydiumSwap.findRaydiumPoolInfo(tokenInAddress, tokenOutAddress)
        if (!poolInfo) {
            throw new Error("Couldn't find the pool info")
        }
        console.log('Found pool info')
        const directionIn = poolInfo.quoteMint.toString() == tokenOutAddress
        const result = await raydiumSwap.calcAmountOut(poolInfo, amount, slippage, directionIn);
        // const { minAmountOut, amountIn } = await raydiumSwap.calcAmountOut(poolInfo, amount, slippage, directionIn)
        res.send(ApiResponse.success(result.minAmountOut.toFixed(6), '操作成功'));
    } catch (e) {
        console.log(e)
        res.send(ApiResponse.error(500, '网络异常，请重试'));
    }
});

app.post('/sol/sendSpl', async (req: { body: any; }, res: { send: (arg0: any) => void; }) => {
    try {
        // 解析请求体中的 JSON 数据
        const data = req.body;
        console.log(data)
        if (isEmpty(data)) {
            res.send(ApiResponse.error(500, '请输入json格式数据'))
            return
        }
        if (data.toWallet == null || data.splToken == null
            || data.privateKey == null || data.amount == null) {
            res.send(ApiResponse.error(500, '输入参数错误'));
            return
        }
        const raydiumSwap = new RaydiumSwap(RPC_URL, data.privateKey)
        console.log(`Raydium swap initialized`)
        // 处理数据...
        const result = await raydiumSwap.sendSolanaToken(data.privateKey, data.toWallet, data.splToken, data.amount);
        res.send(ApiResponse.success(result, '操作成功'))
    } catch (e) {
        res.send(ApiResponse.error(500, '网络异常，请重试'));
    }
});

app.post('/sol/getSplToken', async (req: { body: any; }, res: { send: (arg0: any) => void; }) => {
    // 解析请求体中的 JSON 数据
    const data = req.body;
    console.log(data)
    if (isEmpty(data)) {
        res.send(ApiResponse.error(500, '请输入json格式数据'))
        return
    }
    if (data.walletAddress == null || data.splToken == null) {
        res.send(ApiResponse.error(500, '输入参数错误'));
        return
    }
    // 处理数据...
    const result = await getSplToken(data.walletAddress, data.splToken);
    res.send(ApiResponse.success(result, '操作成功'))
});

async function getSplToken(toAddress: string, mintToken: string) {
    const toPublicKey = new PublicKey(toAddress);
    // const destMint: PublicKey = new PublicKey("YOUR TOKEN ADDRESS");
    // Mint 与要设置或验证的帐户关联（合约地址）
    const tokenM = new PublicKey(mintToken)
    // As this isn't atomic, it's possible others can create associated accounts meanwhile.
    const associatedToken = getAssociatedTokenAddressSync(
        tokenM,
        toPublicKey,
    );
    // console.log(associatedToken.toBase58())
    return associatedToken.toBase58();
}

function isEmpty(obj: {}) {
    return Object.keys(obj).length === 0;
}

async function swap(RPC_URL: string, tokenAAmount: number, tokenInAddress: string, tokenOutAddress: string, privateKey: string, slippage: number, priorityFee: number) {
    const executeSwap = true // Change to true to execute swap
    const useVersionedTransaction = true // Use versioned transaction

    const raydiumSwap = new RaydiumSwap(RPC_URL, privateKey)
    console.log(`Raydium swap initialized`)

    // Loading with pool keys from https://api.raydium.io/v2/sdk/liquidity/mainnet.json
    await raydiumSwap.loadPoolKeys()
    console.log(`Loaded pool keys`)

    // Trying to find pool info in the json we loaded earlier and by comparing baseMint and tokenBAddress
    let poolInfo = raydiumSwap.findPoolInfoForTokens(tokenInAddress, tokenOutAddress)
    if (!poolInfo) {
        console.error('Pool info not found');
        return '';
    } else {
        console.log('Found pool info');
    }
    // if (!poolInfo) poolInfo = await raydiumSwap.findRaydiumPoolInfo(tokenInAddress, tokenOutAddress)
    // if (!poolInfo) {
    //     throw new Error("Couldn't find the pool info")
    // }
    // console.log('Found pool info', poolInfo)
    // console.log('Found pool info')

    const tx = await raydiumSwap.createAndSendTransaction(
        tokenOutAddress,
        tokenAAmount,
        poolInfo,
        priorityFee * LAMPORTS_PER_SOL, // Prioritization fee, now set to (0.0005 SOL)
        // useVersionedTransaction,
        'in',
        slippage // Slippage
    )
    console.log(`https://solscan.io/tx/${tx}`)
    return tx
}

app.listen(8081, () => {
    console.log('Server started on port 8081');
});
